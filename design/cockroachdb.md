# CockroachDB — System Design

## Overview

Distributed SQL database. Geo-distributed, strongly consistent, survives datacenter failures.
- PostgreSQL wire protocol (drop-in compatible)
- Single binary, symmetric nodes (no special master node)
- Scales horizontally for reads and writes

---

## Functional Requirements
- Full SQL with ACID transactions
- Serializable isolation (SSI default)
- Survive: disk, machine, rack, datacenter failures
- Horizontal scale-out reads and writes

## Non-Functional Requirements
- Strong consistency (linearizable within a transaction)
- < 10ms p99 for single-region point lookups
- Multi-region with configurable locality constraints
- No external dependencies (ZooKeeper, etc.)

---

## Architecture Layers

```
┌─────────────────────────────────────────┐
│            SQL Layer (pgwire)           │ ← every node is a SQL gateway
├─────────────────────────────────────────┤
│         DistSQL Engine                  │ ← distributed query execution
├─────────────────────────────────────────┤
│    Transaction Layer (SSI + HLC)        │ ← write intents, timestamp cache
├─────────────────────────────────────────┤
│    Distribution Layer (Ranges)          │ ← meta1/meta2, range routing
├─────────────────────────────────────────┤
│    Replication Layer (Raft per Range)   │ ← consensus, lease holder
├─────────────────────────────────────────┤
│    Storage Layer (RocksDB + MVCC)       │ ← versioned KV, compaction
└─────────────────────────────────────────┘
```

---

## Data Model

**Single monolithic sorted map**: `byte_key → byte_value`

Physically split into **Ranges** (~512 MB each), each backed by RocksDB.

### SQL → KV Encoding
```
Row key:   /<db_id>/<table_id>/<primary_key>/<column_family_id>
Row value: encoded column data

Index key: /<db_id>/<table_id>/<index_id>/<indexed_col_values>
Index val: pointer to PK prefix
```

### Two-Level Range Metadata (4EB addressable)
```
meta1 range (Range 0): key → meta2 range location
meta2 ranges:          key → actual data range location
data ranges:           actual user data

Lookup: meta1 → meta2 → data = 3 RPCs max (cached by client)
```

---

## Replication — Raft per Range

- Each range = independent Raft group (default 3 replicas, N=2F+1)
- **Range lease**: separate from Raft leadership; lease holder serves reads locally (no consensus round-trip)
- Lease holder + Raft leader kept co-located (self-correcting)
- **Coalesced heartbeats**: O(nodes) not O(ranges) — one heartbeat batch per node pair

### Liveness
- **Node liveness table**: each node renews `(epoch, expiration)` entry
- Expired + epoch incremented → another node acquires the lease
- Node offline >5 min → declared dead → replicas rebalanced

---

## Transactions — Lock-Free Distributed Commit

No 2PC coordinator. Uses **write intents + MVCC + transaction record**.

```
BEGIN:
  Assign start_ts (HLC), random priority
  Write PENDING transaction record

WRITE key=K, value=V:
  Write intent: MVCC entry at (K, start_ts) flagged as uncommitted
  Intent contains: txn_id, priority

COMMIT:
  Update transaction record → COMMITTED  ← atomic commit point
  Return to client immediately
  Async: remove intent flags (correctness doesn't depend on cleanup speed)
```

### Conflict Resolution
| Scenario | Action |
|----------|--------|
| Read hits intent with far-future ts | No conflict — read older version |
| Read hits intent within uncertainty window | Restart with future timestamp |
| Read hits intent, writer lower priority | Push writer's commit timestamp forward |
| Writer hits lower-priority intent | Abort conflicting tx |
| Writer hits higher-priority intent | Retry with backoff |
| Writer hits more recent committed value | Restart, advance timestamp |
| Writer hits key with recent read (ts cache) | SSI forces restart |

**Priority system**: random at start; higher priority wins; prevents starvation.

---

## Isolation: SSI (Serializable Snapshot Isolation)

Default isolation level. Eliminates write skew without locking.

- Each tx reads from MVCC snapshot at `start_ts`
- **Timestamp cache**: per range, tracks `key_range → latest_read_ts`
- Writers check: if write_ts < cached_read_ts → restart (another tx already read this key)
- Readers check: if intent present with conflicting read → priority-based resolution

**Clock uncertainty handling**:
```
Tx at ts=T may have missed writes with ts in (T, T+ε)
→ On encountering value in uncertainty window: restart at ts just above conflict
→ Mark node "certain" → at most 1 restart per node touched
→ ε = max clock offset (default 500ms in CockroachDB)
```

---

## Hybrid Logical Clock (HLC)

- Physical component (≈ wall clock) + logical tiebreaker
- Always moves forward, tracks causality, bounded skew
- Every RPC carries sender's HLC → receiver advances on receipt

See [concepts/time_ordering/hybrid_logical_clock.md](../concepts/time_ordering/hybrid_logical_clock.md)

---

## Distributed SQL (DistSQL)

Query execution pushed to data nodes — avoids shipping all data to gateway.

```
SQL query → logical plan (DAG of operators)
         → physical plan (operators placed on lease holders)

Operators:
  TABLE READER → placed on lease holder of each range
  JOIN / AGG   → placed on node with most relevant data
  FINAL AGG    → gateway node

Data flows via gRPC streams between operators
```

- Enables: push-down filters, parallel aggregation, local UPDATE/DELETE
- Each node is a full SQL gateway — client can connect to any node

---

## Capacity Estimation

### Write QPS
- Single-range throughput: ~2k–5k writes/sec (Raft overhead)
- Scale: total_write_QPS / writes_per_range → min ranges needed

### Storage
```
raw_storage  = rows × avg_row_size × replication_factor (3)
mvcc_overhead = ~20% (versioned values, metadata)
total        = raw_storage × 1.2 × 3
```

### Range Count
```
ranges = total_data_size / 512MB
nodes_needed = ranges × 3 replicas / replicas_per_node
              (typical: 50–200 ranges per node)
```

### IOPS
```
Write IOPS (per node, Raft leader):
  Raft WAL:      sequential write per proposal = write_QPS sequential IOPS
  RocksDB write: write_QPS random IOPS (LSM MemTable flush + compaction)
  Intent write:  1 extra write per tx (intent + commit record)
  Example: 5k writes/s/node → 10k IOPS (WAL + RocksDB) + 20% compaction = ~12k IOPS/node
  NVMe SSD: 500k IOPS → headroom for 40× more writes per node

Read IOPS (lease holder):
  Strong read (from lease holder): 0 disk IOPS if RocksDB block cache hit (~99%)
  Cache miss: B-tree height ~3 levels → 3 random IOPS per miss
  Assume 1% miss: read_QPS × 1% × 3 = ~30 IOPS per 1k read QPS
```

### Server Count
```
Example: 1M row table, 10 KB avg row, 5k write QPS, 50k read QPS

Total data:     1M × 10 KB × 1.2 (MVCC) × 3 (replicas) = 36 GB → small
Ranges:         36 GB / 512 MB = 72 ranges × 3 replicas = 216 replica-ranges
Nodes (200/node): ceil(216 / 200) = 2 nodes minimum; run 3 for fault tolerance

At 50k write QPS (total cluster):
  50k ÷ 3 nodes = 17k writes/node → ~34k IOPS/node (WAL + LSM)
  NVMe handles it; run 3–5 nodes to stay under 50% IOPS capacity

Node sizing: 16–32 cores, 64–128 GB RAM, 2× NVMe SSD, 25 Gbps NIC
```

### Latency
| Operation | Typical |
|-----------|---------|
| Single-region point read (lease holder) | <1 ms |
| Single-region write (Raft quorum) | 2–5 ms |
| Cross-region write | RTT/2 (e.g., 50–100 ms US→EU) |
| SSI restart (high contention) | +1 round trip |

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| Intent-based commit (not 2PC) | No coordinator blocking; single write = commit |
| HLC over TSO | No single-node timestamp bottleneck; bounded skew |
| Raft per range | Independent consensus groups → parallel writes |
| Range lease separate from Raft leader | Serve reads locally without consensus |
| Coalesced heartbeats | O(nodes) not O(ranges) heartbeat traffic |
| Two-level meta indirection | Scale key lookup to 4 exabytes |
| SSI default | Strong consistency without the 2PL blocking overhead |

---

## Interview Scenarios

### "Write latency spikes during cross-region transactions"
- Cross-region write = RTT/2 (e.g., 50ms US→EU) per Raft round-trip
- Fix: locality-aware table placement — use `LOCALITY REGIONAL BY ROW` (CockroachDB feature) → each row has a preferred region; writes commit locally
- Avoid cross-region 2PC: design schema so related rows live in same region
- For global tables with low write contention: global table configuration → single region owns writes; others serve reads

### "Hot range — one range (key range) handling all writes (monotonic ID hot spot)"
- Monotonic auto-increment PKs concentrate all inserts in the last range
- Fix: use UUID v4 or ULIDs as primary key → random distribution across ranges
- Or: hash-prefix encoding (CockroachDB `HASH SHARDED INDEXES`) → distributes sequential inserts across N buckets
- Monitor: CockroachDB Console shows `QPS per range`; a range at 10× average = hot spot

### "SSI causes too many transaction restarts under high contention"
- SSI restarts happen when write conflicts with a recent read (timestamp cache hit)
- Fix: prioritized retry with exponential backoff; high-priority transactions win conflicts
- For predictable contention (counter increments): use `SELECT FOR UPDATE` to serialize explicitly → converts SSI to pessimistic locking for that row
- Reduce contention by design: avoid single-row hotspots (use sharded counters, aggregate writes)

### "Read latency from non-leaseholder nodes"
- Read from non-leaseholder: must forward to leaseholder → extra network hop
- Fix: route reads to leaseholder explicitly (CockroachDB client libraries do this automatically via node map)
- For stale reads: `AS OF SYSTEM TIME '-10s'` → served by any follower, no leaseholder needed → 1ms instead of 5ms for cross-region reads
- Follower reads: enable with `SET CLUSTER SETTING kv.closed_timestamps.target_duration = '3s'`

### "Need to survive 2 simultaneous datacenter failures (5-region setup)"
- Default: 3 replicas, survive 1 failure
- Fix: 5 replicas across 5 regions; Raft quorum = 3/5 → survive any 2 failures
- Write latency: must wait for 3rd ACK across regions → increases p99; use `ZONE CONSTRAINT` to prefer nearby replicas for quorum
- Survival goals: `ALTER DATABASE ... SURVIVE REGION FAILURE` (CockroachDB) handles replica placement automatically

### "Constraint: run CockroachDB on fewer nodes to reduce cost"
- Minimum viable CockroachDB: 3 nodes (Raft quorum = 2)
- Each node hosts multiple ranges; default 200 ranges/node is fine; tune `range_max_bytes` if needed
- Reduce storage: enable column compression; use appropriate storage class per data tier
- Right-size nodes: profile actual IOPS per node before sizing — CockroachDB is CPU-bound for SQL, not always I/O-bound

### "Traffic grows 10× — need to scale without downtime"
- Add nodes: CockroachDB rebalances ranges automatically → no manual sharding required
- Scale reads: increase replica count per range → more leaseholders → more read throughput
- Scale writes: more ranges → more range leaders → more write throughput (horizontal partitioning)
- Zero-downtime: rolling node addition; rebalancer runs in background; client retries handle `MOVED` gracefully
