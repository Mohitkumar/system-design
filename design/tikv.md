# TiKV — System Design

## Overview

Distributed transactional key-value store. Storage layer for TiDB (MySQL-compatible SQL).
- Written in Rust
- CNCF graduated project
- Range-based sharding with Multi-Raft replication
- Percolator-based distributed MVCC transactions

---

## Functional Requirements
- Ordered key-value store with Get, Put, Delete, Scan, BatchGet
- ACID transactions across arbitrary keys/regions
- Snapshot isolation (default), linearizable single-key reads
- Horizontal scalability for both reads and writes

## Non-Functional Requirements
- Strong consistency per region (Raft majority)
- External consistency via TSO ordering
- Survive: node failures, network partitions (minority)
- Low tail latency for point lookups

---

## Architecture

```
┌──────────────────────────────────────────┐
│         TiDB / Client Application        │
├──────────────────────────────────────────┤
│           TiKV Client (Rust/Go/Java)     │ ← region cache, retry, routing
├──────────────────────────────────────────┤
│     Transaction Layer (Percolator 2PC)   │ ← start_ts, prewrite, commit
├──────────────────────────────────────────┤
│       Multi-Raft Layer (per Region)      │ ← one Raft group per region
├──────────────────────────────────────────┤
│    RocksDB Storage (4 Column Families)   │ ← CF_DEFAULT, CF_LOCK, CF_WRITE, CF_RAFT
├──────────────────────────────────────────┤
│    Placement Driver (PD) — 3-node Raft   │ ← TSO, metadata, scheduling
└──────────────────────────────────────────┘
```

---

## Data Model

**Flat ordered byte key → byte value namespace**

All keys globally sorted. Region = contiguous key range `[start_key, end_key)`.

### SQL → KV Encoding (TiDB)
```
Table row:   t{tableID}_r{rowID}   → encoded column values
Index entry: t{tableID}_i{indexID}{indexedCols} → rowID
```

---

## Sharding — Region Model

| Property | Value |
|----------|-------|
| Unit | Region = `[start, end)` key range |
| Default size | 96 MB |
| Replication | 3 replicas per region (Raft group) |
| Split trigger | Size > threshold OR write hotspot |
| Merge trigger | Size + QPS both below threshold |

- Range-based (not hash) → range scans stay local
- Adding nodes → regions redistribute → linear scale-out
- Tradeoff: monotonically increasing keys (auto-increment IDs) → hot-spot one region

---

## Replication — Multi-Raft

One independent Raft consensus group per region.

```
PD (Placement Driver)
  ├─ Region A: Leader=Node1, Follower=Node2, Follower=Node3
  ├─ Region B: Leader=Node2, Follower=Node1, Follower=Node3
  └─ Region C: Leader=Node3, Follower=Node1, Follower=Node2
```

- Write: committed only after quorum (2/3) ACK
- **Read Index** optimization: linearizable read without log write — leader confirms quorum still valid, then serves locally
- **Pipelining + batching**: multiple proposals in-flight before ACK
- PD balances leaders across nodes to avoid hot nodes

See [concepts/consensus/multi_raft.md](../concepts/consensus/multi_raft.md)

---

## Storage — RocksDB Column Families

| CF | Key | Value | Purpose |
|----|-----|-------|---------|
| `CF_DEFAULT` | `(key, start_ts)` desc | value bytes | MVCC data versions |
| `CF_LOCK` | `key` | lock + primary ref + TTL | In-flight locks |
| `CF_WRITE` | `(key, commit_ts)` desc | `{type, start_ts}` | Committed version index |
| `CF_RAFT` | raft log keys | log entries | Raft WAL |

Timestamps in **descending order** → forward scan = newest version first.

RocksDB features leveraged:
- **Prefix Bloom Filters** → fast MVCC point lookups
- **IngestExternalFile** → atomic fast snapshot apply
- **CompactRange** → GC of stale MVCC versions
- **DeleteFilesInRange** → fast region deletion on split/merge

---

## Transactions — Percolator Protocol

Two-phase commit without a coordinator. Primary lock = commit authority.

### Prewrite
```
1. Get start_ts from TSO
2. For each write key:
   - Check CF_WRITE for newer committed version → conflict → abort
   - Check CF_LOCK for any lock → conflict → wait/abort
   - Write CF_DEFAULT[(key, start_ts)] = value
   - Write CF_LOCK[key] = {start_ts, primary_key, ttl}
```

### Commit
```
1. Get commit_ts from TSO
2. Verify CF_LOCK[primary_key] still exists
3. Write CF_WRITE[(primary_key, commit_ts)] = {Put, start_ts}  ← COMMIT POINT
4. Delete CF_LOCK[primary_key]
5. Async: commit all secondary keys (any node can do this)
```

### Lock Conflict Resolution (on read)
```
Primary committed?  → commit dangling secondary → read value
Primary TTL expired? → rollback primary + secondaries → read
Primary still live?  → wait / back off
```

See [concepts/consistency/percolator_protocol.md](../concepts/consistency/percolator_protocol.md)

---

## Timestamp Oracle (TSO)

- Embedded in Placement Driver (PD)
- `64-bit = [46-bit physical ms | 18-bit logical counter]`
- Pre-allocates ranges to disk; serves from memory
- Clients batch TSO requests per RPC — amortizes cost
- PD itself is a 3-node Raft group → HA

---

## Placement Driver (PD)

Central brain — 3-node Raft group.

| Function | Details |
|----------|---------|
| TSO | Monotonic timestamps for all transactions |
| Region metadata | Maps all regions → node locations |
| Scheduling | Balance replica count, leader distribution |
| Split/merge triggers | Based on region heartbeats (size + QPS) |
| GC safe point | Tracks min(active start_ts) for MVCC GC |
| Store heartbeats | Liveness, capacity, load monitoring |

---

## Capacity Estimation

### Write QPS
```
avg write QPS = DAU × writes_per_user / 86,400
peak write QPS = avg × 2–3 (80:20 rule)
regions needed = peak_QPS / ~10k writes_per_region_per_sec
```

### Storage
```
raw_data       = rows × avg_row_size
mvcc_overhead  = ~30% (versions in CF_DEFAULT + CF_WRITE entries)
replication    = 3×
total_storage  = raw_data × 1.3 × 3
```

### TSO Bottleneck
```
TSO capacity = ~1M timestamps/sec (batched)
max concurrent txs = TSO_capacity / 2 (2 TSO calls per tx)
→ ~500k txs/sec ceiling before PD becomes bottleneck
```

### IOPS
```
Write path (per TiKV node):
  Raft WAL:       sequential write per proposal = write_QPS sequential IOPS
  CF_DEFAULT:     1 write per key version (LSM MemTable → SSTable flush)
  CF_LOCK:        1 write (prewrite) + 1 delete (commit) = 2 IOPS per tx key
  CF_WRITE:       1 write per committed key
  Total writes:   ~4 RocksDB writes per transactional key write
  Example: 10k tx/s × 5 keys/tx = 50k key writes/s × 4 = 200k IOPS/node
  NVMe (500k IOPS): one node handles; compaction adds ~2× burst → use 2× NVMe

Read path:
  Prefix Bloom Filter: eliminates most unnecessary SSTable reads (in-memory)
  Cache hit (block cache): 0 disk IOPS
  Cache miss: scan CF_WRITE then CF_DEFAULT → 2–4 random IOs per miss
  At 1% miss rate on 10k reads/s: 100 × 3 = 300 random IOPS — negligible
```

### Server Count
```
Example: 1B rows, 100 B avg row, 10k write tx/s, 50k read QPS

Raw data:    1B × 100 B = 100 GB × 3 replicas = 300 GB
Regions:     300 GB / 96 MB = ~3,125 regions × 3 = 9,375 replica-regions
TiKV nodes:  each node ~500 regions → ceil(9,375 / 500) = 19 → run 21 nodes (multiple of 3)
Write IOPS:  10k tx × 5 keys × 4 writes = 200k IOPS ÷ 21 nodes = ~10k IOPS/node ✓

PD cluster:  3 nodes (Raft quorum, TSO + metadata)
TiDB nodes:  SQL layer, stateless; 50k QPS ÷ 5k/server = 10 nodes

Node sizing (TiKV): 16–32 cores, 128 GB RAM, 2× 2TB NVMe, 25 Gbps NIC
```

### Latency
| Operation | Typical |
|-----------|---------|
| Point read (CF_WRITE + CF_DEFAULT) | 1–5 ms |
| Single-region write (Raft quorum) | 2–10 ms |
| Cross-region distributed tx (2PC) | 20–50 ms |
| TSO call (batched) | <1 ms |

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| Range-based regions (not hash) | Range scans stay local; split is metadata-only |
| Raft per region (Multi-Raft) | Independent consensus → parallel writes, no global bottleneck |
| Percolator 2PC | No coordinator; any node resolves stale locks; primary lock = source of truth |
| TSO in PD | Strict ordering without HLC uncertainty; simpler correctness proof |
| 4 Column Families in RocksDB | Separate MVCC data/locks/commits/raft → targeted compaction, GC, bloom filters |
| Descending timestamp order | Newest version first on RocksDB scan → fast read of latest value |
| Read Index | Linearizable reads without Raft log write → lower read amplification |

---

## Interview Scenarios

### "Hot region — one region handling all writes (auto-increment primary key)"
- Range-based sharding: monotonic IDs all map to the same region's key range
- Fix: use UUID or hash-prefix encoding for primary key → distributes writes across many regions
- TiDB: `SHARD_ROW_ID_BITS` shard the implicit row ID → N regions handle inserts simultaneously
- PD detects hot regions via heartbeat QPS metrics and automatically splits + redistributes

### "TSO becomes a bottleneck at high transaction rate"
- TSO ceiling: ~500k transactions/sec (batched); beyond this PD becomes bottleneck
- Fix: reduce TSO calls — read-only transactions use `ts_bound` (stale read from follower, no TSO needed)
- Batch TSO requests: clients already batch by default; increase batch size under high load
- Horizontal TSO: TiDB 6.0+ introduced local TSO for specific transaction scopes (reduce global TSO load)

### "Transaction spans many regions — high latency"
- Percolator 2PC: prewrite to all regions → get commit_ts → commit primary → async commit secondaries
- Cross-region transaction: each RPC is a Raft round-trip (~5–10ms per region hop)
- Fix: co-locate related data in same region — use composite key `(user_id, entity_id)` so user's data lives in one region range
- Async commit (TiKV 5.0+): commit point is prewrite completion + async secondary cleanup → cuts commit latency by ~50%

### "Read latency too high — followers serving stale data"
- Default: reads go to region leader (linearizable); followers may be 100ms behind
- Stale read: tolerate bounded staleness → read from any follower within `max_staleness` window → eliminates leader RTT
- Read index: leader confirms quorum still holds, then serves read locally without Raft log write → linearizable at lower cost
- Block cache tuning: increase RocksDB block cache (default 45% RAM) → cache hit rate 99%+ → near-zero disk IOPS on reads

### "Region count grows to millions — PD scheduling overhead"
- PD stores all region metadata; millions of regions = large memory footprint on PD nodes
- Typical: 1 TiKV node holds 500–1000 regions; 1000 nodes × 500 regions = 500k regions → PD handles fine
- At 1M+ regions: increase PD node RAM; tune heartbeat interval (`split-merge-interval`) to reduce PD gossip volume
- Region merge: idle small regions below QPS + size threshold are auto-merged by PD → reduces region count

### "Constraint: require exactly-once semantics for financial transactions"
- Percolator provides ACID with snapshot isolation by default
- For exactly-once: include idempotency key in transaction; before prewrite, check if key already committed in CF_WRITE
- Application layer: include `payment_id` as a key in the transaction; conditional write fails if already exists → safe retry
- 2PC failure recovery: if primary lock has commit_ts in CF_WRITE, transaction committed; safe to return success even on retry

### "Traffic grows 10× — scale TiKV cluster"
- Add TiKV nodes: PD detects imbalance → schedules region migration to new nodes automatically
- Region split: hot regions auto-split by PD when QPS or size threshold exceeded
- No manual resharding: TiKV's range-based sharding + PD auto-rebalance handles scale-out transparently
- TiDB SQL nodes: stateless → add horizontally behind load balancer; no coordination needed
- PD cluster: stays at 3 nodes (Raft); TSO throughput scales with batching, not node count
