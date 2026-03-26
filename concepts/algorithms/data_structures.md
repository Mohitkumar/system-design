# Data Structures in Databases

---

## Skip List

**Structure**: multiple layers of linked lists; each layer skips over more elements.
```
L3: 1 ─────────────────────── 50 ─────────── 100
L2: 1 ──────── 20 ─────────── 50 ─── 75 ──── 100
L1: 1 ── 10 ── 20 ── 30 ──── 50 ── 60 ── 75 ─ 100
L0: 1 ─ 5 ─ 10 ─ 15 ─ 20 ─ 25 ─ 30 ... (all elements)
```

- Insert/search/delete: O(log n) average
- **Lock-free concurrent variant**: compare-and-swap on each level pointer
- Ordered → supports range scans naturally
- Simpler to implement than B-trees for in-memory use

**Used in**: RocksDB/LevelDB MemTable, Redis sorted sets (ZSET), CockroachDB/TiKV in-memory write buffer, Java `ConcurrentSkipListMap`

---

## B+ Tree

- All data in leaf nodes; internal nodes are routing keys only
- Leaf nodes linked as doubly-linked list → range scans in O(1) seek + O(k) scan
- Height 3–4 for billions of rows (branching factor 100–1000)
- In-place updates: read page → modify → write page (random I/O)

**Used in**: PostgreSQL, MySQL InnoDB (clustered), SQLite, Oracle — default index structure for relational DBs

---

## LSM Tree (Log-Structured Merge-Tree)

- MemTable (skip list) → immutable MemTable → SSTables on disk
- Writes always append → sequential I/O
- Reads merge across multiple levels; bloom filters skip irrelevant SSTables
- Compaction: merge-sort SSTables periodically to reclaim space

**Used in**: RocksDB, LevelDB, Cassandra, HBase, ScyllaDB, TiKV (CF_DEFAULT), InfluxDB

---

## Bloom Filter

**Structure**: bit array + k hash functions. Test set membership.
- Insert: set k bits at positions `h1(x), h2(x), ..., hk(x)`
- Query: check all k positions — all set → *probably* present; any unset → *definitely* absent
- **False positives possible; false negatives impossible**
- Space: ~10 bits/element for 1% false positive rate

**Used in**: RocksDB/Cassandra (skip SSTable reads), BigTable, Akamai CDN cache, Google Chrome safe-browsing, distributed caches, Ethereum (transaction log filtering)

---

## Hash Table

- O(1) average insert/lookup via hash function
- Open addressing (linear/quadratic probing) or chaining
- No ordering → no range queries
- Must fit in memory (or use extendible/linear hashing for disk)

**Used in**: Redis all data structures, PostgreSQL hash joins, MySQL hash indexes, in-memory join hash tables in any RDBMS, HashMap in query execution engines

---

## Inverted Index

**Structure**: term → sorted list of document IDs (posting list)
```
"database" → [doc1, doc5, doc12, doc99]
"engine"   → [doc1, doc3, doc12]
```
- Intersection of posting lists = AND query
- Union = OR query
- TF-IDF / BM25 scores stored alongside doc IDs for ranking
- Compressed posting lists (delta encoding + varint)

**Used in**: Elasticsearch, Apache Lucene, Solr, PostgreSQL `GIN` index (full-text search), Sphinx, Meilisearch

---

## R-Tree (Spatial Index)

- Hierarchical bounding rectangles around spatial objects
- Each node = minimum bounding rectangle (MBR) of all children
- Query: intersect query rectangle with MBRs, recurse only into overlapping children
- Supports: point-in-polygon, nearest neighbor, range queries on 2D/3D data
- Variants: R*-tree (better splits), Hilbert R-tree (packed)

**Used in**: PostGIS (PostgreSQL), MySQL spatial indexes, MongoDB 2d/2dsphere, SQLite R-tree extension, GIS systems (ESRI, QGIS)

---

## Trie / Radix Tree (Patricia Tree)

- Character-by-character key tree; common prefixes share nodes
- Radix tree = compressed trie (skip single-child nodes)
- Lookup: O(m) where m = key length
- Memory-efficient for string keys with shared prefixes
- Supports: prefix search, autocomplete, IP prefix matching

**Used in**: Linux kernel routing table (LPC-trie), Nginx routing, Redis prefix commands, database dictionary compression, IP CIDR routing (BGP)

---

## Roaring Bitmap

- Compressed bitmap for sets of 32-bit integers
- Key space divided into 2^16 containers; each container is:
  - Array (< 4096 elements) → sorted uint16 array
  - Bitmap (≥ 4096 elements) → 8KB bitset
  - Run-length encoded (long runs)
- Intersection/union: O(n/64) with SIMD
- 2–10× more compact than plain bitmaps

**Used in**: Elasticsearch (doc filtering), Apache Druid/Spark/Pinot (column bitmaps), ClickHouse, Pilosa, Lucene posting lists

---

## HyperLogLog (HLL)

- Approximate count-distinct with O(log log n) memory
- Error: ~1.6% with 1.5 KB; ~0.8% with 12 KB
- Algorithm: hash each element, observe max leading zeros in hash → estimate cardinality
- Mergeable across shards

**Used in**: Redis `PFADD`/`PFCOUNT`, PostgreSQL `pg_hyperloglog`, BigQuery COUNT(DISTINCT), Snowflake approximate_count_distinct, Presto/Trino, distributed analytics

---

## Count-Min Sketch

- Approximate frequency counter; 2D array of counters + multiple hash functions
- Insert: increment d positions (one per hash function row)
- Query: return min of d positions → upper bound of true frequency
- Fixed memory regardless of distinct elements; no deletion

**Used in**: Apache Storm (top-K heavy hitters), network traffic analysis, Apache Flink, database query optimizers (frequency estimation for join order), spam filters

---

## Merkle Tree

- Binary tree where each node = hash of its children
- Leaf nodes = hash of data blocks
- Root hash = fingerprint of entire dataset
- Detect difference between two trees: O(log n) by comparing root, then drilling down

**Used in**: Git (object store), Bitcoin/Ethereum (transaction trees), ZooKeeper (data verification), Cassandra anti-entropy (comparing replica state), IPFS, Dynamo (replica reconciliation)

---

## T-Digest

- Approximate quantile estimation (median, p99, p999)
- Maintains sorted list of centroids (mean + count); merges nearby centroids
- More accurate at extremes (tails) where precision matters most
- Mergeable across nodes

**Used in**: Elasticsearch percentile aggregations, Prometheus (histogram\_quantile approximation), Netflix Atlas, DataDog, Kibana

---

## Consistent Hash Ring

- Nodes placed on virtual ring (0 to 2^32)
- Key → hash → clockwise walk → first node = owner
- Virtual nodes: each physical node has multiple positions → even distribution
- Node add/remove: only neighboring keys move

**Used in**: Amazon DynamoDB, Apache Cassandra, Redis Cluster, Memcached (ketama), Akamai CDN routing

---

## Fenwick Tree (Binary Indexed Tree)

- O(log n) prefix sum queries and point updates
- More cache-friendly than segment tree
- Classic use: order statistics, rank computation

**Used in**: database internal sort statistics, columnar analytics (cumulative distribution), competitive DB benchmarking tools

---

## Summary Table

| Data Structure | Lookup | Range Scan | Space | Used For |
|---------------|--------|-----------|-------|----------|
| B+ Tree | O(log n) | O(log n + k) | Medium | Disk indexes, RDBMS |
| Skip List | O(log n) | O(log n + k) | Medium | In-memory write buffer |
| Hash Table | O(1) | ✗ | Low | Point lookup, joins |
| LSM Tree | O(log n)* | O(log n + k)* | + overhead | Write-heavy storage |
| Inverted Index | O(1) lookup | Merge lists | Large | Full-text search |
| R-Tree | O(log n) | O(log n + k) | Medium | Spatial/geo queries |
| Trie/Radix | O(m) | O(m + k) | Low-Med | Prefix, routing |
| Bloom Filter | O(k) | ✗ | Very low | Existence check |
| HyperLogLog | O(1) | ✗ | ~1 KB | Count distinct |
| Count-Min | O(k) | ✗ | Fixed | Frequency estimate |
| Merkle Tree | O(log n) | ✗ | O(n) | Diff/verification |
| Roaring Bitmap | O(1) | O(n/64) | Very low | Set intersection |

*with bloom filter optimization
