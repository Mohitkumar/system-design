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
