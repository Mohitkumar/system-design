# Storage Engines

> Key reference: *Designing Data-Intensive Applications* (DDIA) by Martin Kleppmann — Chapters 3 (Storage & Retrieval) and 7 (Transactions / MVCC).

Two dominant storage engine architectures: **B+ Tree** (read-optimized) and **LSM Tree** (write-optimized).

---

## B+ Tree

### How It Works

A balanced N-ary tree. Internal nodes hold keys as routing guides; **all data lives in leaf nodes**. Leaf nodes are linked as a doubly-linked list for range scans.

```
                    [  30  |  60  ]               ← internal node (keys only)
                   /        |        \
          [10|20]        [40|50]       [70|80]     ← internal nodes
          /  |  \        /  |  \       /  |  \
        [.] [.] [.]    [.] [.] [.]  [.] [.] [.]  ← leaf nodes (key + value/ptr)
         ↔   ↔   ↔  ↔   ↔   ↔   ↔  ↔   ↔   ↔    ← linked list for range scan
```

- **Page size** = 4–16 KB (aligned to OS page / block device sector).
- **Branching factor** = 100–1000 keys per internal node → tree height 3–4 for billions of rows.
- **Height 3 tree, factor 1000:** 1000^3 = 1 billion leaf entries reachable in 3 I/Os.

> DDIA p.80: *"The underlying write operation of B-trees is to overwrite a page on disk with new data... In contrast, log-structured indexes only append to files."*

### Write Path (B+ Tree)

```
INSERT key=42
  │
  ├─ 1. WAL append (crash safety) ──────────────────► disk (sequential)
  │
  ├─ 2. Root-to-leaf traversal (3–4 page reads)
  │       └─ each page: check buffer pool → cache hit (RAM) or cache miss (disk read)
  │
  ├─ 3. Modify leaf page in buffer pool (in-memory)
  │
  ├─ 4. If page full → SPLIT:
  │       promote median key to parent → may cascade up to root
  │       allocate new page → write both pages to disk
  │
  └─ 5. Mark pages dirty → background flusher writes to disk
```

**Write amplification:** 1 logical write → potentially many page reads + rewrites (splits, parent updates). Random I/O to disk.

### Read Path (B+ Tree)

```
SELECT WHERE key=42
  │
  ├─ 1. Root page → check buffer pool
  │       HIT  → read from RAM (nanoseconds)
  │       MISS → read from disk into buffer pool (milliseconds)
  │                └─ OS read-ahead: prefetch N sequential pages speculatively
  │
  ├─ 2. Walk internal nodes (height 3–4), each step = 1 page
  │
  ├─ 3. Leaf page found → return row / pointer to heap
  │
  └─ 4. Range scan: follow leaf linked list until end of range
```

**Buffer pool hit rate** is critical. 99% hit rate = most reads served from RAM. Miss = physical disk read (random I/O, ~1–5ms SSD, ~10ms HDD).

### OS Page Alignment

```
Disk block size: 512B or 4KB (physical sector)
OS page size:    4KB
DB page size:    4KB–16KB (multiple of OS page)

Aligned read:    request aligns to page boundary → single syscall, no wasted I/O
Unaligned read:  spans two pages → two reads, extra copy → wasted I/O
```

B+ tree pages are deliberately sized and aligned to OS pages. PostgreSQL uses 8KB pages; InnoDB uses 16KB.

### Examples

| DB | Engine | Notes |
|---|---|---|
| PostgreSQL | Heap + B+ tree indexes | Heap stores rows; indexes point to heap TIDs |
| MySQL InnoDB | Clustered B+ tree | PK is the table (clustered); secondary indexes point to PK |
| SQLite | B+ tree | Single file; entire DB is one B-tree |
| Oracle | B+ tree | Industry standard; bitmap indexes also available |

---

## LSM Tree (Log-Structured Merge-Tree)

> DDIA p.78: *"Log-structured storage engines... turn random-access writes into sequential writes on disk, which enables higher write throughput."*

### How It Works

Never overwrites in place. All writes are appends. Periodically merge sorted files.

**Components:**
- **WAL** — append-only crash recovery log on disk.
- **MemTable** — in-memory sorted structure (red-black tree or skip list). Active write buffer.
- **Immutable MemTable** — MemTable being flushed; still readable.
- **SSTables** — immutable sorted files on disk, organized in levels.
- **Bloom Filter** — per-SSTable: *"does this key definitely NOT exist here?"* — skips unnecessary disk reads. False positives possible; false negatives impossible.
- **Sparse Index** — per-SSTable: maps key ranges to byte offsets. One entry per block (~4KB). Binary search → seek to block → scan within block.

### Write Path (LSM Tree)

```
PUT key=42, value=X
  │
  ├─ 1. Append to WAL ──────────────────────────────► disk (sequential, fast)
  │
  ├─ 2. Insert into MemTable (in-memory sorted map)
  │       └─ return OK to client ← write complete from client's perspective
  │
  ├─ 3. MemTable reaches threshold (e.g. 64 MB) → become Immutable MemTable
  │       New MemTable created for incoming writes
  │
  └─ 4. Background: flush Immutable MemTable → SSTable on disk (Level 0)
          └─ SSTable = sorted key-value pairs + sparse index + bloom filter

  Levels on disk:
    L0: [SST_a][SST_b][SST_c]          ← freshly flushed, may overlap, ~4 files
    L1: [─────────────SST──────────]   ← compacted, non-overlapping, 10× larger
    L2: [────────────────────────────] ← 10× larger than L1
    L3: [────────────────────────────]
```

**Compaction:** Merge SSTables from L0 → L1 → L2. Deduplicates keys (keep newest version), removes tombstones. Keeps each level sorted and non-overlapping (except L0).

**Write amplification:** Each byte may be rewritten `O(L)` times across levels (typically 10–30×). Trade: sequential I/O vs B+ tree's random I/O → still much faster writes.

### Read Path (LSM Tree)

```
GET key=42
  │
  ├─ 1. Check MemTable ────────────────────── HIT → return (nanoseconds)
  │
  ├─ 2. Check Immutable MemTable ─────────── HIT → return
  │
  ├─ 3. For each SSTable, newest first (L0 → L1 → ...):
  │       a. Bloom Filter
  │           NEGATIVE → key definitely absent → SKIP (save disk I/O)
  │           POSITIVE → maybe present → continue
  │       b. Binary search sparse index → find block offset
  │       c. Check block cache (buffer pool)
  │           HIT  → read from RAM
  │           MISS → physical disk read (see path below)
  │       d. Scan block for key
  │           FOUND → return value (check if tombstone)
  │           NOT FOUND → continue to next SSTable
  │
  └─ 4. Not found anywhere → key does not exist
```

**Worst case read:** touches all SSTables across all levels. Bloom filters and block cache make this rare in practice.

### Physical Read Path (Cache Miss Detail)

```
Application: get(key)
    │
    ▼
DB Block Cache (LRU in-memory, e.g. 8GB RocksDB block cache)
    ├─ HIT  → return block ────────────────────────────── ~100ns
    └─ MISS
         │
         ▼
    OS Page Cache (kernel buffer cache, typically GBs of RAM)
         ├─ PAGE HIT  → copy to user-space ──────────── ~1µs
         └─ PAGE MISS → issue block device request
              │
              ▼
         Block Device (SSD / NVMe / HDD)
              │  read 4KB aligned block from physical location
              │  NVMe SSD → ~50–100µs
              │  HDD      → ~5–10ms (seek + rotational latency)
              │
              │  OS read-ahead: prefetch N sequential blocks speculatively
              │  (helpful for range scans; wasteful for random point lookups)
              │
              ▼
         Data returns up the stack:
         disk → OS page cache → DB block cache → storage engine → app
```

**Cache hit ratio matters above all else.**
- 99% hit → 1 in 100 requests hits disk → fast
- 95% hit → 5 in 100 requests hit disk → 5× more I/O → noticeable degradation
- 80% hit → catastrophic for a latency-sensitive system

### LSM Compaction Strategies

| Strategy | How | Best for |
|---|---|---|
| **Size-tiered** (default Cassandra) | Merge SSTables of similar size together | Write-heavy; fewer compactions |
| **Leveled** (RocksDB default) | Strict level sizes; L(n+1) = 10× L(n) | Read-heavy; bounded read amplification |
| **FIFO** | Evict oldest SSTables when size limit hit | Time-series, logs; no merging |
| **Tiered + Leveled** | Tiered at top levels, leveled at bottom | Hybrid workloads (RocksDB universal) |

### Examples

| DB | Notes |
|---|---|
| RocksDB | Facebook; underlying engine for TiKV, CockroachDB, MyRocks |
| Cassandra | LSM per node; pluggable compaction |
| LevelDB | Google's reference implementation |
| HBase | LSM on top of HDFS |
| ScyllaDB | C++ reimplementation of Cassandra |

---

## Indexing Strategies

### Clustered Index

Data rows physically stored **in index order**. The index IS the table.

```
Clustered index on user_id:
  Leaf page 1: [id=1, name=Alice, ...] [id=2, name=Bob, ...] [id=3, ...]
  Leaf page 2: [id=4, ...] ...

Range scan WHERE user_id BETWEEN 1 AND 100:
  → sequential leaf page reads → very fast (data is contiguous on disk)
```

- Only **one clustered index per table** (data can only be in one physical order).
- InnoDB: clustered on primary key by default.
- PostgreSQL: heap-based (unordered); `CLUSTER` command reorders once but doesn't stay ordered.

### Secondary Index

A separate B+ tree whose leaf nodes store a **pointer back to the primary row**.

```
Secondary index on email:
  Leaf: [email="alice@x.com" → PK=42] [email="bob@x.com" → PK=7] ...

Query: SELECT * WHERE email="alice@x.com"
  1. Lookup secondary index → get PK=42
  2. Lookup clustered index with PK=42 → fetch full row  ← "key lookup" / double read
```

- **Index-only scan / covering index:** if query only needs columns inside the index, skip step 2.
- In LSM (Cassandra): secondary indexes are a separate local SSTable structure per node.

### Hash Index

In-memory hash map: `key → byte offset on disk`. Used by Bitcask (Riak's engine).

```
hash("alice") → offset 4096
hash("bob")   → offset 8192
```

- O(1) point lookup.
- **No range queries** — hash has no ordering.
- Entire hash map must fit in RAM.
- **Used by:** Bitcask, Redis (all data structures backed by hash tables), InnoDB adaptive hash index (auto-built for hot B+ tree pages).

> DDIA p.73: *"The hash table must fit in memory... Also, range queries are not efficient."*

### Composite Index

Index on multiple columns `(a, b, c)` — **left-prefix rule**:

```
Effective for: (a), (a,b), (a,b,c)
Useless for:   (b), (c), (b,c)   ← can't skip leading column
```

Column order: most selective or most frequently filtered column leftmost.

---

## LSM Indexing: Local vs Global (Distributed)

Relevant in distributed LSM stores (Cassandra, HBase, DynamoDB).

### Local (Secondary) Index

Each node indexes only the data it owns.

```
Node 1 owns keys A–M → local secondary index covers A–M only
Node 2 owns keys N–Z → local secondary index covers N–Z only

Write on Node 1: update local index only → fast, no coordination

Read on secondary key "email":
  → scatter request to ALL nodes
  → each checks its local index
  → coordinator merges results
```

- Write: fast (local, no coordination).
- Read: scatter-gather → fan-out to every node → higher latency, more load.
- **Used by:** Cassandra secondary indexes.

### Global Index

A separate index structure, itself partitioned across nodes, independent of data partitioning.

```
Data partitioned by user_id (hash).
Global index on email, partitioned by hash(email).

Write user_id=42, email="alice@x.com":
  → write data to Node A (hash(42))
  → write index entry to Node B (hash("alice@x.com"))  ← async or 2-phase

Read WHERE email="alice@x.com":
  1. Route to index node for hash("alice") → get user_id=42   ← 1 hop
  2. Route to data node for hash(42) → return row             ← 1 hop
```

- Write: must update data node + index node → distributed write (2PC or async, risk of inconsistency).
- Read: 2 targeted hops → faster than scatter-gather.
- **Used by:** DynamoDB GSI, Google Spanner, CockroachDB.

### Range Queries in LSM

```
WHERE key BETWEEN a AND b:
  1. MemTable: iterate from a to b (skip list → O(log n) to find a, then scan)
  2. For each SSTable:
     a. Bloom filter useless (only checks point existence)
     b. Sparse index: find first block where key ≥ a
     c. Scan blocks sequentially until key > b
  3. K-way merge across all sources, deduplicate by version timestamp
```

Range queries are worse in LSM than B+ tree because:
- L0 SSTables may overlap → must check all of them.
- K-way merge overhead across levels.
- After compaction, L1+ are sorted + non-overlapping → range scans are efficient there.

---

## MVCC (Multi-Version Concurrency Control)

> DDIA p.239: *"A key principle of snapshot isolation is readers never block writers, and writers never block readers."*

### The Problem

Without MVCC: reader takes a shared lock → blocks writer. Writer takes exclusive lock → blocks reader. Serialized access kills throughput.

### How It Works

Every write creates a **new version** of a row tagged with a transaction ID (txn_id). Old versions are kept until no active transaction needs them.

```
Initial state:
  key="balance" → [txn_id=1, value=100]

T2 starts read (takes snapshot at txn_id=2):
  sees value=100

T3 writes balance=200 (txn_id=3):
  key="balance" → [txn_id=3, value=200]   ← new version
                  [txn_id=1, value=100]   ← old version kept

T2 still reading:
  its snapshot = txn_id=2 → ignores txn_id=3 version → sees 100  ← consistent!

T2 commits → txn_id=1 version now eligible for GC (VACUUM in PostgreSQL)
```

### Visibility Rule

Transaction with `snapshot_id = S` sees version `V` if:
- `V.txn_id ≤ S` (committed before snapshot)
- `V` is not superseded by another version with `txn_id ≤ S`

### MVCC in B+ Tree (PostgreSQL)

```
Heap page stores multiple tuples (versions) for the same logical row:
  [xmin=1, xmax=3, value=100]   ← visible to txns 1–2; dead after T3
  [xmin=3, xmax=∞, value=200]   ← visible to txns ≥ 3; current

Index still points to both heap tuples.
VACUUM: removes dead tuples, reclaims space, updates index pointers.
HOT update (Heap-Only Tuple): if updated columns are not in any index →
  new tuple on same page → no index update needed → faster writes.
```

### MVCC in LSM (RocksDB / Cassandra)

LSM is naturally multi-versioned — every write is an append:

```
key="balance" @ts=3 → 200   ← newest
key="balance" @ts=1 → 100   ← older version

Read at snapshot ts=2:
  scan SSTables newest-first, find latest version with ts ≤ 2 → returns 100

Compaction = GC:
  when no active snapshot needs ts=1 → drop it during compaction
```

### Isolation Levels via MVCC

| Level | What you see | Anomalies prevented |
|---|---|---|
| Read Uncommitted | Latest (even uncommitted) | Nothing |
| Read Committed | Latest committed per statement | Dirty reads |
| Repeatable Read | Snapshot at txn start | Dirty + non-repeatable reads |
| Serializable | As if serial execution | All anomalies (phantom reads etc.) |

MVCC naturally provides **Snapshot Isolation** (Repeatable Read). True Serializable requires SSI — Serializable Snapshot Isolation (PostgreSQL uses this since 9.1).

---

## B+ Tree vs LSM Tree — Summary

| Dimension | B+ Tree | LSM Tree |
|---|---|---|
| Write I/O | Random (in-place page updates) | Sequential (append WAL + flush) |
| Read I/O | Low (tree height = 3–4 pages) | Medium–High (multi-level SSTable scan) |
| Write amplification | Medium | High (compaction rewrites data L× times) |
| Read amplification | Low | Medium (bloom filters + cache reduce it) |
| Space amplification | Low | Medium (stale versions until compaction) |
| Range scans | Excellent (leaf linked list) | Good at lower levels; worse at L0 |
| Compaction overhead | None | Background CPU + I/O (can spike latency) |
| Best for | Read-heavy OLTP, mixed workloads | Write-heavy: logs, time-series, events |
| Examples | PostgreSQL, MySQL, SQLite, Oracle | Cassandra, RocksDB, HBase, LevelDB |

> DDIA p.84: *"As a rule of thumb, LSM-trees are typically faster for writes, whereas B-trees are thought to be faster for reads. Reads are typically slower on LSM-trees because they have to check several different data structures and SSTables at different stages of compaction."*

---

## Read/Write Path Diagrams

### B+ Tree Write Path

```
Client write
     │
     ▼
  WAL append ──────────────────────────────────────► Disk (sequential)
     │
     ▼
Buffer Pool (in-memory pages)
  ┌─ page already in pool? ─┐
  │ YES                     │ NO
  ▼                         ▼
dirty-mark             read page from disk into pool (random I/O)
  │                         │
  └──────────┬──────────────┘
             ▼
  Modify leaf page in pool
             │
             ▼
  Page full? ──YES──► SPLIT: allocate new page, write both ──► Disk (random)
             │
             NO
             ▼
  Background page flusher ──────────────────────────────────► Disk (random)
```

### LSM Tree Write Path

```
Client write
     │
     ▼
  WAL append ──────────────────────────────────────► Disk (sequential, fast)
     │
     ▼
  MemTable (sorted skip list, in-memory)
  return OK to client ← write done
     │
     ▼ (64MB threshold)
  Immutable MemTable ──background flush──► SSTable L0 ──► Disk (sequential)
                                                │
                                                ▼ (L0 has ≥4 files)
                                        Compaction: L0 → L1 ──► Disk (sequential)
                                        Compaction: L1 → L2 ──► Disk
```

### LSM Tree Read Path

```
Client read (GET key)
     │
     ▼
MemTable ──HIT──► return (~100ns)
     │
     MISS
     ▼
Immutable MemTable ──HIT──► return
     │
     MISS
     ▼
SSTable L0 (newest first, may overlap → check all):
  │ Bloom Filter ──NEGATIVE──► SKIP
  └─ POSITIVE
       │ Sparse Index → block offset
       ▼
     DB Block Cache ──HIT──► return (~100ns)
       │
       MISS
       ▼
     OS Page Cache ──HIT──► return (~1µs)
       │
       MISS
       ▼
     Disk (NVMe ~50µs, HDD ~10ms)
     read-ahead: prefetch adjacent blocks
       │
       ▼
     populate OS page cache → DB block cache → return value

     NOT FOUND in L0? → repeat for L1 → L2 → ... → key not exist
```
