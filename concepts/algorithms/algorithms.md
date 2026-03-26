# Algorithms in Databases & Distributed Systems

---

## Sorting

### External Merge Sort
- Sort data too large to fit in RAM
- Phase 1: load chunks, sort in-memory → write sorted **runs** to disk
- Phase 2: k-way merge of runs using min-heap
- I/O: O(N log N / B) where B = buffer size
- Used in: PostgreSQL sort node, MySQL ORDER BY without index, Hadoop MapReduce shuffle, Spark sort-based shuffle, database bulk load

### TimSort
- Hybrid merge sort + insertion sort
- Detects and exploits natural runs in data
- O(n log n) worst, O(n) best (already sorted)
- Used in: Python `sorted()`, Java `Arrays.sort()`, Pandas — any in-memory sort in query engines

### Introsort
- Hybrid: quicksort + heapsort (fallback on deep recursion) + insertion sort (small arrays)
- O(n log n) guaranteed; cache-friendly in practice
- Used in: C++ `std::sort`, .NET, query engine in-memory sorts

---

## Searching

### Binary Search
- O(log n) on sorted array
- Used everywhere: SSTable sparse index lookup, B-tree key search within page, range boundary detection, PostgreSQL binary search on sorted pages

### Interpolation Search
- O(log log n) for uniformly distributed data
- Estimate position: `pos = lo + (key - arr[lo]) / (arr[hi] - arr[lo]) * (hi - lo)`
- Used in: database index statistics, histogram-based query optimizer estimates

### Ternary Search
- Find max/min of unimodal function
- Used in: query plan cost optimization, gradient descent in ML-based query optimizers

---

## Hashing

### MurmurHash / xxHash
- Non-cryptographic; fast, good distribution
- MurmurHash3: 128-bit output, ~1 GB/s throughput
- xxHash3: faster (~8–10 GB/s), SIMD-optimized
- Used in: partitioning (Cassandra, Kafka), Bloom filters, hash joins, in-memory hash tables

### Consistent Hashing
- Map keys + nodes to a ring; key goes to first clockwise node
- Virtual nodes: each physical node has V positions → even distribution
- Node join/leave: O(K/N) keys move (K=keys, N=nodes)
- Used in: DynamoDB, Cassandra, Redis Cluster, load balancers

### Rendezvous Hashing (Highest Random Weight)
- Each key independently scores all nodes: `score(key, node) = hash(key + node)`
- Key → node with highest score
- No ring needed; naturally handles node add/remove
- Used in: CDN cache routing, Nginx upstream selection, distributed caches

---

## String / Pattern Matching

### Aho-Corasick
- Multi-pattern string matching in O(n + m + z) where z = matches
- Build failure links (like KMP but for multiple patterns)
- Used in: intrusion detection (Snort), full-text multi-keyword search, spam filters, grep `-F` (multiple fixed strings)

### Knuth-Morris-Pratt (KMP)
- Single pattern matching in O(n + m) — no backtracking
- Prefix function avoids redundant comparisons
- Used in: `strstr()` implementations, LIKE queries in DBs, grep

### Boyer-Moore
- Pattern matching from right to left; skips large portions of text
- O(n/m) best case — faster than KMP in practice for long patterns
- Used in: PostgreSQL LIKE with `pg_trgm`, grep, text editors find/replace

### Trigram Index
- Index all 3-character substrings of text
- LIKE `%term%` → find trigrams of `term` → intersect posting lists
- Used in: PostgreSQL `pg_trgm` extension, Elasticsearch n-gram tokenizer

---

## Graph Traversal

### BFS (Breadth-First Search)
- Level-by-level traversal using a queue
- O(V + E), finds **shortest path** in unweighted graphs
- Used in: Neo4j shortest path, social network friend-of-friend queries, network topology discovery, web crawlers

### DFS (Depth-First Search)
- Recursive/stack-based; explores as deep as possible
- O(V + E)
- Used in: cycle detection, topological sort (dependency resolution), tree serialization, CockroachDB range metadata traversal

### Dijkstra's Algorithm
- Shortest path in weighted graphs (non-negative weights)
- O((V + E) log V) with binary heap
- Used in: routing tables (OSPF), mapping/navigation (Google Maps), CDN routing, network distance estimation

### A* Search
- Dijkstra + heuristic function h(n) to guide search toward goal
- Optimal if h(n) is admissible (never overestimates)
- Used in: game pathfinding, GPS navigation, robot motion planning

### PageRank
- Iterative: score(v) = sum of score(u)/out_degree(u) for all u → v
- Converges to stationary distribution of random walk
- O(iterations × E)
- Used in: Google search ranking, LinkedIn network ranking, Apache Spark GraphX, citation analysis

### Topological Sort (Kahn's / DFS)
- Linear ordering of DAG vertices
- O(V + E)
- Used in: build system dependency resolution (Bazel, Make), SQL query plan ordering, schema migration dependency ordering, task scheduling

---

## Compression

### LZ4
- Byte-oriented LZ compression; extremely fast decode (~1–2 GB/s)
- Lower compression ratio than Zstd but minimal CPU
- Used in: RocksDB block compression (default option), Kafka message compression, ClickHouse, network packet compression where CPU is scarce

### Snappy (Google)
- Similar to LZ4; ~250–500 MB/s compress, ~500 MB/s decompress
- Used in: LevelDB/RocksDB, Cassandra, Hadoop, Parquet column pages

### Zstandard (Zstd)
- Much better ratio than LZ4/Snappy; fast decode with dictionary
- Tunable: level 1 = near LZ4 speed, level 19 = near gzip ratio
- Used in: PostgreSQL 15+ WAL compression, Zookeeper snapshots, Linux kernel, Facebook's internal storage

### Delta + RLE Encoding (Columnar)
- **Delta**: store differences between consecutive values → small ints → better compression
- **RLE**: runs of same value stored as (value, count)
- Used in: Parquet, ORC, ClickHouse MergeTree (sort key columns), Apache Druid (timestamp columns)

### Gorilla Encoding (Time-Series)
- XOR consecutive floats; leading/trailing zeros encoded in 1–2 bytes
- ~1.37 bytes/point on typical metrics data
- Used in: Facebook Gorilla TSDB, Prometheus (Cortex), VictoriaMetrics, InfluxDB IOx

---

## Cache Eviction

### LRU (Least Recently Used)
- Evict the item not accessed for the longest time
- Implementation: doubly-linked list + hash map → O(1) access + eviction
- Used in: OS page cache, PostgreSQL buffer pool, Redis (approximation), CDN edge caches

### LFU (Least Frequently Used)
- Evict the item accessed fewest times
- Implementation: frequency buckets + min-heap → O(log n) or O(1) with frequency list trick
- Better than LRU for stable hot-key workloads
- Used in: Redis (LFU policy), CPU L2/L3 caches, CDN static asset caches

### Clock (Second Chance)
- Circular buffer; each page has a reference bit
- On eviction: scan clockwise; if bit=1 → clear and skip; if bit=0 → evict
- O(1) amortized; approximates LRU with less memory
- Used in: Linux page cache (not pure Clock but similar), OS virtual memory managers

### TinyLFU / W-TinyLFU
- Window (small LRU) + Protected (frequent LRU) + Probationary segments
- Count-Min sketch for frequency tracking
- Near-optimal hit rate; O(1) operations
- Used in: Caffeine (Java cache library), used by Spring, Hadoop, Cassandra read cache

---

## Query Optimization

### Cost-Based Optimizer (CBO)
- Enumerate possible plans (join order, index choice, scan method)
- Estimate cost using statistics (row counts, cardinality, histograms)
- Choose minimum cost plan
- Used in: PostgreSQL planner, MySQL optimizer, CockroachDB, Spark Catalyst

### Dynamic Programming for Join Order
- Optimal join ordering is NP-hard (all permutations)
- DP: memoize optimal plan for each subset of relations
- O(3^n) time and O(2^n) space — practical up to ~20 tables
- Used in: PostgreSQL join planner, Oracle query optimizer

### Predicate Pushdown
- Move WHERE conditions as close to data source as possible
- Reduces rows early → less data through pipeline
- Used in: Spark Catalyst, PostgreSQL, Presto/Trino, Parquet reader (row group filtering)

### Vectorized Execution (SIMD)
- Process a batch (vector) of rows per operator call instead of one row at a time
- SIMD instructions process 4–16 values in one CPU instruction
- 2–10× speedup on analytical queries
- Used in: DuckDB, ClickHouse, Snowflake, Apache Arrow, Velox

---

## Concurrency Control

### MVCC (Multi-Version Concurrency Control)
- Every write = new version with timestamp; readers see consistent snapshot
- Readers never block writers; writers never block readers
- GC removes old versions below oldest active transaction
- Used in: PostgreSQL, MySQL InnoDB, CockroachDB, TiKV, Oracle

### Two-Phase Locking (2PL)
- Growing phase: acquire locks; shrinking phase: release locks
- Strict 2PL: hold write locks until commit (prevents cascading aborts)
- Deadlock detection via wait-for graph (cycle = deadlock)
- Used in: MySQL InnoDB (default), SQL Server, DB2

### OCC (Optimistic Concurrency Control)
- Execute without locks; validate at commit time
- Abort if any read was modified by a committed transaction
- Low overhead under low contention; high abort rate under high contention
- Used in: CockroachDB SSI, TiKV Percolator, Hekaton (SQL Server in-memory)

---

## Encoding / Serialization

### Varint (Variable-length Integer)
- Small integers use fewer bytes: 0–127 = 1 byte, 128–16383 = 2 bytes
- Used in: Protocol Buffers, RocksDB keys, Lucene posting lists, Parquet

### Zigzag Encoding
- Maps signed integers to unsigned: 0→0, -1→1, 1→2, -2→3
- Enables varint for negative numbers
- Used in: Protocol Buffers sint32/sint64, Avro, Thrift

### Frame of Reference (FOR) / PFOR
- Subtract min value from all integers in a block; encode deltas with fewer bits
- PFOR: most values fit in b bits; outliers stored separately
- Used in: Lucene doc IDs, Elasticsearch, columnar OLAP engines

---

## Distributed Algorithms

### Gossip Protocol
- Each node periodically picks random peers and exchanges state
- Epidemic spreading: O(log N) rounds to reach all N nodes
- Scales to thousands of nodes; no SPOF
- Used in: Cassandra membership, Consul health checks, Bitcoin peer discovery, DynamoDB ring state

### Phi Accrual Failure Detector
- Instead of binary alive/dead, outputs suspicion level φ
- φ computed from heartbeat inter-arrival distribution (exponential/normal)
- Threshold tunable: higher φ threshold = fewer false positives, slower detection
- Used in: Cassandra, Akka clustering

### Raft Log Replication
- Leader appends entry → sends AppendEntries to followers → commits on majority ACK
- Safety: never commit entries from previous terms unless current-term entry also committed
- Used in: etcd, CockroachDB, TiKV, Consul

### Consistent Snapshot (Chandy-Lamport)
- Distributed snapshot without stopping the system
- Each process records own state; markers propagate through channels
- Used in: Apache Flink checkpointing, distributed debuggers, global state capture

---

## Summary: Algorithm → System Mapping

| Algorithm / DS | Systems |
|---------------|---------|
| Skip list | RocksDB MemTable, Redis ZSET, TiKV write buffer |
| B+ Tree | PostgreSQL, MySQL, SQLite, Oracle |
| Bloom filter | RocksDB, Cassandra, BigTable, Chrome |
| Inverted index | Elasticsearch, Lucene, PostgreSQL GIN |
| R-Tree | PostGIS, MongoDB geo, MySQL spatial |
| Consistent hashing | DynamoDB, Cassandra, Redis Cluster |
| HyperLogLog | Redis, BigQuery, Snowflake |
| Roaring bitmap | Druid, ClickHouse, Elasticsearch |
| External merge sort | PostgreSQL, MySQL, Spark shuffle |
| Vectorized execution | DuckDB, ClickHouse, Snowflake |
| Gossip | Cassandra, Bitcoin, Consul |
| Raft | etcd, CockroachDB, TiKV |
| MVCC | PostgreSQL, MySQL, CockroachDB, TiKV |
| W-TinyLFU | Caffeine, Cassandra read cache |
| Gorilla encoding | Prometheus, VictoriaMetrics, InfluxDB |
