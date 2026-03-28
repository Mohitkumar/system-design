# Distributed Search Engine

**Examples:** Elasticsearch, Apache Solr, Google Search, Bing

---

## Concepts

### Inverted Index
Core data structure mapping term → list of document IDs (posting list).

```
Document 1: "the quick brown fox"
Document 2: "the quick rabbit"
Document 3: "brown rabbit"

Inverted Index:
  "brown" → [1, 3]
  "fox"   → [1]
  "quick" → [1, 2]
  "rabbit"→ [2, 3]
  "the"   → [1, 2]
```

Each posting list entry can also store: position, frequency, field, score metadata.

### Document vs Term Partitioning

| | Document Partitioning | Term Partitioning |
|--|---|---|
| **How** | Each shard owns a disjoint set of documents; full inverted index for those docs | Each shard owns a range of terms; posting list for that term lives on one shard |
| **Write (index)** | Route doc to its shard → shard builds its local inverted index | Each term in the doc → update its term's shard → N network writes per doc |
| **Read (search)** | Scatter to ALL shards → each returns top-K → merge (scatter/gather) | Route query terms to their shards → exact posting lists → no scatter |
| **Partial failure** | Missing shard → incomplete results (recall drops) | Missing shard → terms on it unavailable → query may fail |
| **Hot spots** | Balanced by doc count | Hot terms (e.g. "the") overwhelm their shard |
| **Used by** | Elasticsearch, Solr, Google | Rare in practice; used in some academic IR systems |

**Production default: document partitioning** — scatter/gather is manageable; hot-term shards are not.

---

## Functional Requirements

- `crawl(seed_urls)` — discover and fetch documents from the web
- `index(document)` — parse, analyze, and add document to inverted index
- `search(query, filters?, page, size)` → ranked list of results with snippets
- `delete(doc_id)` — remove document from index
- Full-text search with relevance ranking (TF-IDF / BM25)
- Filters: date range, category, language, domain

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Search latency | < 200ms p99 |
| Index latency | < 30s from crawl to searchable |
| Availability | 99.99% for search; 99.9% for indexing |
| Consistency | Eventual — new docs visible within seconds |
| Scale | 10B documents, 10K queries/sec |
| Durability | No data loss on node failure |

---

## Capacity Estimation

**Assumptions:** 10B docs, avg doc size 10 KB, 10K QPS peak search, 1K docs/sec indexing rate.

**Storage:**
```
Raw docs (S3):      10B × 10 KB = 100 TB
Inverted index:     ~30% of raw (terms + posting lists) = 30 TB
  per shard (20):   30 TB / 20 = 1.5 TB/shard
Forward index:      ~20% of raw (doc metadata, snippets) = 20 TB
Total index nodes:  30 TB / 500 GB per node = 60 index nodes
Replication (3×):   60 × 3 = 180 index node-replicas
```

**QPS:**
```
80:20 rule: 10K QPS peak; avg = 10K × 0.2 / 0.8 ≈ 2.5K QPS
Scatter/gather: each query fans out to all 20 shards → 10K × 20 = 200K shard-queries/sec
Per shard: 200K / 20 = 10K shard-queries/sec (each shard handles 10K QPS)
```

**IOPS:**
```
Search (read):
  Index stored in memory (hot) + SSD (warm)
  Cache hit rate ~90% → 10% miss → 10K × 20 shards × 10% = 20K random disk reads/sec
  Per shard: 1K random IOPS → NVMe (500K IOPS) handles easily

Indexing (write):
  1K docs/sec × ~200 terms/doc = 200K posting list updates/sec (in-memory segment)
  Segment flush to disk: every 1s → ~200 MB/s sequential write per indexing node
  NVMe sequential (1 GB/s) → 5 indexing nodes handle load
```

**Server count:**

| Role | Count | Sizing |
|---|---|---|
| Crawler nodes | 20 | 8 cores, 16 GB RAM, 1 Gbps NIC |
| Indexer nodes | 5 | 16 cores, 64 GB RAM, 2× NVMe |
| Index/search nodes | 60 (20 shards × 3 replicas) | 16 cores, 128 GB RAM, 2× 2TB NVMe |
| Cluster manager | 3 (Raft quorum) | 8 cores, 32 GB RAM |
| Query router | 5 (stateless) | 8 cores, 16 GB RAM |
| S3 (raw doc store) | Managed | — |

---

## High-Level Architecture

```
                    ┌─────────────┐
Seed URLs ──►       │   Crawler   │ ──► Raw Docs ──► S3 (raw store)
                    └─────────────┘                      │
                                                         ▼
                                                   ┌──────────┐
                                                   │  Indexer  │  (reads from S3,
                                                   └──────────┘   writes to index shards)
                                                         │
                          ┌──────────────────────────────┤
                          ▼          ▼          ▼        ▼
                      [Shard 0]  [Shard 1]  [Shard 2] ... [Shard N]
                      primary    primary    primary       primary
                      replica1   replica1   replica1      replica1
                      replica2   replica2   replica2      replica2
                          │          │          │
                          └──────────┴──────────┘
                                     │
                         ┌──────────────────────┐
User Query ──► Router ──►│   Cluster Manager    │──► scatter ──► merge ──► ranked results
                         │  (shard map, merge)  │
                         └──────────────────────┘
```

---

## Component Deep Dive

### 1. Crawler

```
Seed URLs → URL Frontier (priority queue) → Fetcher → HTML Parser
                ▲                                           │
                └─────── extracted links ──────────────────┘
                                                  │
                                             Raw Doc Store (S3)
                                             + URL seen set (Bloom filter)
```

- **URL Frontier**: priority queue (BFS + politeness delay per domain); persisted to avoid re-crawl on restart
- **Politeness**: max 1 req/sec per domain; respect `robots.txt`; distributed frontier to avoid thundering herd
- **Dedup**: SHA-256 of content → skip near-duplicates (SimHash for fuzzy dedup)
- **Freshness**: re-crawl schedule based on change frequency (high-traffic pages: hours; static pages: weeks)
- **Scale**: 20 crawlers × 500 pages/sec = 10K pages/sec → crawls 10B docs in ~12 days

### 2. Indexer

Reads raw docs from S3, transforms to inverted index entries.

```
Raw Doc (S3)
    │
    ▼
Text Extraction  (HTML → plain text; PDF, DOCX parsers)
    │
    ▼
Analysis Pipeline:
  Tokenizer     → "Quick Foxes" → ["Quick", "Foxes"]
  Lowercasing   → ["quick", "foxes"]
  Stop words    → remove "the", "a", "is"
  Stemming      → ["quick", "fox"]  (Porter/Snowball stemmer)
  N-grams       → ["quick fox"] (optional, for phrase search)
    │
    ▼
Term → DocID mapping → Partition by hash(docID) → write to shard
    │
    ▼
In-memory segment (write buffer)
    │  flush every 1s
    ▼
Immutable segment on disk (Lucene segment / SSTable-like)
    │  background merge
    ▼
Larger merged segments (fewer files → faster search)
```

**MapReduce for bulk indexing:**
```
Map:    doc → emit(term, {docID, position, freq})
Reduce: (term, [{docID,...}]) → sorted posting list for that term
Output: partial inverted index per reducer → loaded into shards
```
Used for initial index build or full reindex. Online indexing uses the streaming pipeline above.

### 3. Index Shard (Storage + Search)

Each shard stores:

| Structure | Purpose |
|---|---|
| **Inverted index** | term → posting list `[(docID, freq, positions)]` |
| **Forward index** | docID → `{title, url, snippet, metadata}` |
| **Term dictionary** | sorted term list + pointer to posting list on disk |
| **Bloom filter** | fast term existence check before disk read |
| **Doc norms** | per-field length normalization for BM25 scoring |

**Segment merge (like LSM compaction):**
```
Write: new docs → in-memory segment
Flush: segment → immutable on-disk segment
Merge: small segments → larger segment (background)
       removes deleted docs, re-sorts posting lists
       reduces file handles + improves read performance
```

### 4. Query Router & Cluster Manager

```
Query arrives at Router:
  1. Parse query: tokenize, stem, handle operators (AND/OR/NOT, phrase "quick fox")
  2. Lookup shard map from Cluster Manager (cached locally, TTL=30s)
  3. Scatter: send sub-query to one replica of each shard in parallel
  4. Gather: collect top-K results per shard (each shard returns scored doc list)
  5. Merge: global top-K by score (merge K sorted lists → heap-based O(K log N))
  6. Fetch snippets: use forward index to generate highlighted snippets
  7. Return ranked results to client
```

**Cluster Manager (etcd/ZooKeeper):**
- Stores shard map: `shard_id → [primary_node, replica1, replica2]`
- Leader election for coordinator role
- Watches node liveness (ephemeral nodes); triggers rebalance on node failure
- Manages replica assignment across AZs for fault isolation

### 5. Ranking — BM25

```
BM25 score(doc D for query Q):
  Σ over each query term t:
    IDF(t) × (freq(t,D) × (k1+1)) / (freq(t,D) + k1×(1 - b + b×|D|/avgDL))

IDF(t)  = log((N - df(t) + 0.5) / (df(t) + 0.5))  ← rare terms score higher
freq    = term frequency in doc
|D|     = doc length; avgDL = average doc length
k1=1.5, b=0.75  (tunable constants)
```

- Computed per shard locally → shard returns top-K `(docID, score)` pairs
- Router merges → global top-K by score

---

## Replication

Document-partitioned shards; each shard = 1 primary + 2 replicas.

```
Index write:  Router → Primary shard
              Primary → replicate to replicas (semi-sync: ack after 1 replica)
              Primary returns success to indexer

Search read:  Router → any replica (round-robin) ← spreads read load
              Primary not needed for search reads
```

**Replica promotion on primary failure:**
1. Cluster manager detects heartbeat miss (3× misses = dead)
2. Elects replica with highest replication offset as new primary
3. Updates shard map in etcd
4. Other replica and router re-read shard map → start routing to new primary

---

## Reindexing — The Hard Problem

Reindexing is needed when: ranking algorithm changes, tokenizer/analyzer upgraded, schema change, corruption recovery.

### Blue-Green Reindex
```
Index v1 (live, serving search)
     │
     ├─ Build Index v2 in parallel (new cluster or new shard set)
     │    - Read all raw docs from S3
     │    - Apply new analysis pipeline
     │    - Takes hours to days for 10B docs
     │
     ├─ Catch up: replay index writes that landed on v1 during v2 build
     │    - Use a write-ahead log / Kafka topic for all index mutations
     │    - Replay from the offset at which v2 build started
     │
     ├─ Validation: run sample queries against v2; compare results
     │
     └─ Atomic cutover: update router's shard map pointer from v1 → v2
          - Instant; no downtime
          - Rollback: point back to v1 (v1 still live until v2 proven stable)
```

### Incremental / Rolling Reindex
- Reindex one shard at a time; take it out of rotation, reindex, bring back
- Search continues on other shards during reindex (reduced recall temporarily)
- Simpler operationally; suitable for minor schema changes

### Segment-Level Reindex
- Add a new field or update scoring: rewrite only affected segments
- Lucene supports `UpdateByQuery` — rewrites matching docs without full rebuild
- Used for lightweight changes (new field, re-score subset of docs)

---

## Failure Modes

| Failure | Impact | Mitigation |
|---|---|---|
| Shard primary crash | That shard's writes stall briefly | Replica promotion < 30s; replicas serve reads during failover |
| Shard fully lost (all replicas) | Documents on that shard not searchable | Reindex from S3 raw store; 3 replicas in different AZs makes this rare |
| Crawler node down | Crawl rate drops | Stateless crawler; URL frontier persisted; other crawlers pick up work |
| Indexer node down | Index lag increases | Stateless indexer; re-read from S3; Kafka consumer group rebalances |
| Cluster manager down | Shard map stale (cached at router) | 3-node Raft cluster; routers cache map with 30s TTL → can serve stale during brief outage |
| Slow shard (tail latency) | Search query waits for slowest shard | Hedged requests: send to 2 replicas, use first response |
| Hot term shard | One shard overwhelmed (term partitioning) | Use document partitioning — avoids term hot spots |

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Partition strategy | Document partitioning (scatter/gather) | Avoids hot-term shards; simpler write path |
| Shard count | 20 shards (fixed at creation) | Over-shard initially; rebalance by moving whole shards |
| Replication | 1 primary + 2 replicas per shard | Read scale + survive 1 AZ failure |
| Search reads | Any replica | Spreads read load; primary not a bottleneck |
| Segment storage | Immutable segments + merge | Like LSM; sequential writes; merge handles compaction |
| Ranking | BM25 per shard + global merge | Standard IR; computed locally to avoid data movement |
| Reindex strategy | Blue-green with WAL replay | Zero-downtime; instant rollback |
| Raw doc store | S3 | Source of truth; enables reindex without re-crawl |

---

## Interview Scenarios

### "Search latency spikes to 2s on some queries"
- Scatter/gather: slowest shard = query latency → one slow shard tanks all queries
- Fix: hedged requests — send each sub-query to 2 replicas, use first response; cancel the other
- Identify slow shard: log per-shard latency; check for large posting lists on hot terms
- Hot terms (e.g., "the", "and"): skip-list on posting list → early termination on common terms

### "Index lag grows — new documents take 10 minutes to appear"
- Bottleneck: indexer throughput or shard write throughput
- Check indexer pipeline: tokenization CPU, S3 read latency, shard write ack time
- Add indexer nodes (stateless — just add more consumers from S3/Kafka)
- Reduce segment flush interval: flush every 100ms instead of 1s → docs visible sooner (near-real-time search)

### "Traffic grows 10× — 100K QPS"
- Scatter/gather fan-out: 100K × 20 shards = 2M shard-queries/sec → 100K per shard
- Add replicas: 20 shards × 5 replicas = 100 nodes serving reads
- Add query routers: stateless → scale horizontally
- Caching: L1 in-process cache on router for popular queries (top 1% of queries = ~50% of traffic)

### "Need to support phrase search — 'quick brown fox' as exact phrase"
- Inverted index stores positions: `"fox" → [(doc1, pos:[4]), (doc2, pos:[7,12])]`
- Phrase query: find docs where positions of consecutive terms are consecutive
- Performance: positional index is larger (~3× posting list size); intersect positions at query time
- Alternative for common phrases: index bigrams/trigrams as separate terms

### "Reindex of 10B documents takes 5 days — too slow"
- Parallelize: 5 days / 5 indexer nodes = 1 day with 25 nodes
- Prioritize: crawl and index high-value docs first (top domains by traffic)
- MapReduce bulk indexing: run on Spark/EMR with S3 as source → write directly to shard storage
- Partial reindex: only reindex docs where analyzer output changed (hash old analysis output; skip if unchanged)

### "Constraint: search must return results even during index shard failure"
- With document partitioning and 20 shards: losing 1 shard means 5% of documents not searchable
- Mitigation: 3 replicas across 3 AZs — probability of all 3 down simultaneously is negligible
- Degraded mode: if shard unavailable after 100ms → return results from other 19 shards with a "partial results" flag
- SLA: define acceptable recall drop (95% recall = 19/20 shards) vs hard failure

### "Constraint: freshness SLA tightens to <5s (near-real-time search)"
- Pipeline today: crawl → S3 → batch indexer → shard → searchable (minutes)
- Fix: streaming pipeline — crawl → Kafka → streaming indexer → shard in-memory segment
- Reduce segment flush: every 100ms → docs visible within 1s of indexing
- Trade-off: more small segments → more merge work → slightly higher read overhead
- Elasticsearch NRT: 1s refresh interval (configurable to 100ms at cost of merge overhead)
