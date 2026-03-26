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
