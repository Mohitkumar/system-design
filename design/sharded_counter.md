# Sharded Counter

**Use cases:** Tweet likes/views/retweets, YouTube view count, Reddit upvotes, product ratings, real-time leaderboards, trending topics (Top-K)

---

## Concepts

### Why Single Counter Fails at Scale

```
Single Redis key:  INCR tweet:123:likes

At 1M writes/sec on one key:
  - Redis INCR is single-threaded → serialized → ~100K ops/sec ceiling per key
  - 1 hot key = 1 Redis slot = 1 shard → no horizontal scale
  - Lock contention in DB: UPDATE likes = likes+1 WHERE tweet_id=123
    → row lock held per write → serialized → ~10K writes/sec max on SQL
```

### Sharded Counter Pattern
```
Write: hash(counter_id + random_shard) → increment one of N shards
Read:  fetch all N shards → SUM

tweet:123:likes:shard_0  →  342,891
tweet:123:likes:shard_1  →  289,443
tweet:123:likes:shard_2  →  401,112
                             ─────────
Total likes              → 1,033,446
```

- Writes spread across N shards → N× throughput
- Reads require aggregation (read amplification = N reads)
- Tradeoff: write throughput ↑↑, read latency ↑ (but reads are cacheable)

### Count-Min Sketch (for Top-K / approximate counts)
```
d hash functions × w counters (d rows × w columns)

Increment(item):
  for each hash function h_i:
    table[i][h_i(item)] += 1

Query(item):
  return min(table[i][h_i(item)] for i in 0..d)

Properties:
  - Never undercounts (only overestimates due to hash collisions)
  - Space: O(d × w) regardless of number of distinct items
  - d=5, w=2000 → error ε < 0.1% with 99% confidence
```

### Top-K with Min-Heap
```
Maintain a min-heap of size K:
  - For each new item count update:
    if item already in heap → update count (re-heapify)
    else if count > heap.min → replace min with new item
  - Heap always holds the K items with highest counts

Space: O(K)  ← independent of total number of items
```

---

## Functional Requirements

- `increment(counter_id, delta=1)` — increment a counter (e.g., tweet view, like)
- `decrement(counter_id, delta=1)` — for unlike, unvote
- `getCount(counter_id)` → approximate total count
- `getTopK(category, K, window)` → top K counters by count in a time window
- Windowed counts: last 1 min, 1 hour, 24 hours, all-time
- Batch increment (for bulk events like video play segments)

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Write throughput | 1M increments/sec per counter (peak) |
| Read latency | < 10ms for getCount; < 100ms for getTopK |
| Count accuracy | ±0.1% for totals; ±1% for Top-K |
| Availability | 99.99% writes (fail open); 99.9% reads |
| Eventual consistency | Count visible within 1–2s |
| Scale | 1B distinct counters (tweets, videos, users) |

---

## Capacity Estimation

**Assumptions:** 500M active counters, 1M increments/sec peak, 100K getCount/sec, 10K getTopK/sec.

**Sharding:**
```
Writes per counter: up to 100K/sec for viral content (celebrity tweet)
Shards per counter: 100K / 50K ops/shard = 2 shards minimum; use 10 for headroom
Total shard-keys: 500M counters × 10 shards = 5B shard keys
```

**Storage (Redis):**
```
Per shard key: counter_id (20B) + shard_id (2B) + int64 value (8B) + overhead = ~60B
Total: 5B keys × 60B = 300 GB → fits in Redis cluster (10 nodes × 32 GB each)
Replication: 3× → 30 Redis nodes total
```

**Storage (persistent DB — Cassandra):**
```
Periodic flush from Redis → Cassandra (every 10s)
Per counter row: counter_id + shard + value + updated_at = ~100B
5B rows × 100B = 500 GB → 5 Cassandra nodes × 128 GB RAM
```

**IOPS:**
```
Write IOPS:
  1M increments/sec → Redis INCR: in-memory, no disk IOPS (AOF: 1 batch/sec)
  Cassandra flush: 1M/10s batching = 100K writes/sec to Cassandra
  Cassandra: 100K × 3 replicas = 300K writes/sec → 5 nodes × 60K IOPS (NVMe)

Read IOPS:
  100K getCount × 10 shards = 1M Redis reads/sec → Redis cluster handles
  Cache hit rate ~95% → Cassandra fallback: 5K reads/sec = negligible
```

**Server count:**

| Role | Count | Sizing |
|---|---|---|
| Counter API nodes | 10 | 16 cores, 32 GB RAM |
| Redis cluster (hot shards) | 30 (10 primary + 20 replicas) | 8 cores, 64 GB RAM |
| Cassandra (durable store) | 5 | 16 cores, 128 GB RAM, 2× NVMe |
| Kafka (event buffer) | 5 | 16 cores, 64 GB RAM, 2× NVMe |
| Top-K aggregator | 3 | 16 cores, 64 GB RAM |

---

## High-Level Architecture

```
Client
  │
  ▼
[Counter API]  ← route increment to shard; aggregate reads
  │                  │
  │ increment        │ getCount
  ▼                  ▼
[Kafka]         [Redis Cluster]  ← hot tier (in-memory shards)
  │              shard_0..shard_N
  │              per counter
  ▼
[Counter Aggregator]  ← consumes Kafka, batches, flushes
  │
  ├──► [Redis]      ← INCR shard keys
  │
  └──► [Cassandra]  ← periodic flush for durability
             │
             ▼
         [Top-K Service]  ← reads Kafka stream, maintains Count-Min Sketch + Min-Heap
             │
             ▼
         [Top-K Cache (Redis)]  ← getTopK served from here
```

---

## Write Path — Sharded Increment

```
Client: increment(tweet:123:likes)

1. API: select shard = rand(0, N-1)  OR  hash(client_id) % N  (sticky for same client)
   key = "c:{counter_id}:{shard}"  e.g. "c:tweet:123:likes:7"

2. Produce to Kafka:  topic=counter-events, key=counter_id (same counter → same partition)
   message: {counter_id, shard_id, delta, timestamp}

3. Counter Aggregator (Kafka consumer):
   Batch events for 100ms window
   For each unique (counter_id, shard_id): sum deltas in batch
   INCR "c:tweet:123:likes:7" by batch_sum  ← single Redis INCR per shard per 100ms

4. Periodic durability flush (every 10s):
   Aggregator reads current shard values from Redis
   Writes to Cassandra: UPDATE counters SET value=? WHERE counter_id=? AND shard=?
```

**Why Kafka in the path:**
- Decouples API from Redis writes → API never blocks on Redis latency
- Batching in aggregator: 1M events/sec → 10K batched Redis INCRs/sec (100× reduction)
- Replayable: if aggregator crashes, replay from Kafka offset (no lost counts)

**Direct path (low-latency option):**
```
For sub-second latency requirement: API directly INCRs Redis (skip Kafka)
  INCR c:{counter_id}:{rand_shard}
  Tradeoff: Redis write load is direct (no batching); higher Redis IOPS
  Use for: like buttons (user expects instant feedback)
```

---

## Read Path — getCount

```
getCount(counter_id):

1. Check read cache: GET "count:{counter_id}" from Redis (TTL=5s)
   → Cache hit (95%): return immediately

2. Cache miss: fetch all shards in parallel
   MGET c:{counter_id}:0, c:{counter_id}:1, ..., c:{counter_id}:9
   → Single Redis round-trip with MGET (pipelined)

3. Sum shard values → total

4. Write back to read cache: SET "count:{counter_id}" total EX 5

Consistency:
  - Shard values updated async (Kafka path, ~100ms lag)
  - Read cache adds 5s TTL → count can be 5s stale
  - Acceptable for likes/views (user doesn't notice 5s lag on counts)
  - For exact count: skip cache; sum shards directly (higher latency)
```

---

## Windowed Counts (last 1h, 24h, all-time)

```
Store counts in time-bucketed shards:

Key format: c:{counter_id}:{shard}:{window_bucket}
  All-time:    c:tweet:123:likes:7:all
  1-hour:      c:tweet:123:likes:7:hour:2024010109   (bucket = YYYYMMDDHH)
  1-minute:    c:tweet:123:likes:7:min:202401010945

getCount(counter_id, window=1h):
  current_hour + previous_hour buckets → MGET all shards × 2 buckets → SUM

Expiry:
  1-minute buckets: TTL = 2 minutes (auto-expire from Redis)
  1-hour buckets:   TTL = 25 hours
  All-time bucket:  no TTL (persisted to Cassandra)
```

---

## Top-K Trending

### Problem
Find top K counters (e.g., trending tweets) by increment rate in last 1 hour.

### Count-Min Sketch + Min-Heap Pipeline

```
Kafka stream of increment events:
  {counter_id: "tweet:456:likes", delta: 1, timestamp: T}

Top-K Aggregator (per time window):

1. Count-Min Sketch (d=5, w=2000):
   For each event: update CMS[i][h_i(counter_id)] += delta

2. Min-Heap of size K:
   Estimated count = min(CMS[i][h_i(counter_id)])
   if counter_id in heap → update count → re-heapify
   elif estimated_count > heap.min → replace heap.min with (counter_id, count)

3. Every 30s: flush Top-K snapshot to Redis
   ZADD trending:1h <score=count> <member=counter_id>
   TTL = 65 minutes on the key

getTopK(category="tweets", K=10, window="1h"):
  ZREVRANGE trending:1h 0 9 WITHSCORES ← Redis sorted set
```

**Multi-window Top-K:**
```
Maintain separate CMS + Heap per window:
  trending:1min  → for breaking news (fast decay)
  trending:1h    → for hourly trends
  trending:24h   → for daily trends

Decay: use exponential moving average on counts:
  score = alpha × new_count + (1 - alpha) × prev_score
  alpha=0.7 for 1min (fast decay), alpha=0.1 for 24h (slow decay)
```

### Distributed Top-K (at scale)

```
Multiple Top-K aggregator nodes (partitioned by counter_id hash):

Each node:
  - Processes its partition of Kafka events
  - Maintains local CMS + top-K heap
  - Publishes local top-K to coordinator every 30s

Coordinator:
  - Receives top-K lists from all aggregators
  - Merges: union of all local top-K → re-rank → global top-K
  - Exact for top-K (false positives from CMS may include non-top-K items, but top-K items are always present)
  - Write to Redis sorted set

Why local top-K is sufficient:
  Top-K items have high counts → they appear in every partition's local top-K
  Low-count items may be missing from some local lists — but they can't be global top-K
```

---

## Accuracy vs Throughput Tradeoffs

| Approach | Accuracy | Write Throughput | Read Latency | Use |
|----------|----------|-----------------|--------------|-----|
| Single counter (Redis INCR) | Exact | ~100K/sec | O(1) | Low-traffic counters |
| Sharded counter (N=10) | Exact | ~1M/sec | O(N) | High-traffic counters |
| Local in-process + async flush | ±seconds of lag | Unlimited | O(N) + cache | Very hot counters |
| Count-Min Sketch | ±0.1% | Unlimited | O(1) | Top-K, approx totals |
| HyperLogLog | ±0.81% | High | O(1) | Unique visitor counts |

---

## Failure Modes

| Failure | Impact | Mitigation |
|---------|---------|------------|
| Redis node crash | Its shard keys unavailable | 3 replicas; replica promotion <30s; read from replica during failover |
| Aggregator crash | Increment lag grows | Kafka offset not committed → replay from last committed offset; no lost counts |
| Kafka partition leader down | Produce fails for that partition | ISR election <10s; API retries with backoff |
| Cassandra unavailable | Durability flush fails | Redis holds counts; aggregator retries flush; Kafka provides replay if Redis also lost |
| Count-Min Sketch OOM | Top-K service restarts from empty | CMS rebuilt from Kafka replay (configurable look-back window) |
| Redis memory full | New INCRs fail | Evict expired window buckets; scale out Redis cluster |

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Shard selection | Random shard on write | Uniform distribution; no coordination |
| Read aggregation | MGET all shards + SUM | Single round-trip (pipelined); O(N) but N=10 is fast |
| Write batching | Kafka → 100ms batch → INCR | 100× Redis write reduction; no data loss on crash |
| Read cache | 5s TTL on total count | Avoids shard scatter on every read; 95%+ hit rate |
| Durability | Cassandra flush every 10s | Redis is fast but ephemeral; Cassandra survives Redis loss |
| Top-K algorithm | Count-Min Sketch + Min-Heap | O(1) update; O(K) space; ±0.1% accuracy; handles 1B distinct items |
| Window buckets | Time-bucketed shard keys with TTL | Auto-expiry; no explicit delete; windowed queries = MGET multi-bucket |
| Shard count N | 10 per counter | Balance read fan-out (10 Redis reads) vs write throughput (10× scale) |

---

## Interview Scenarios

### "Celebrity tweet gets 10M likes in 10 seconds (viral burst)"
- 1M likes/sec → 100K per shard (10 shards) → within Redis INCR ceiling
- Kafka batching: aggregator collapses burst into batches → Redis sees 10K INCRs/sec not 1M
- If burst exceeds: increase N shards for hot counters dynamically (detect via rate monitor → re-shard)
- Read cache shields: 5s TTL means 99%+ of read requests never hit Redis during viral burst

### "Exact count needed — bank balance, inventory (not approximate)"
- Sharded counters introduce read-time aggregation race (shard A incremented, shard B not yet)
- Fix: single shard with Redis INCR (serialized, exact) — only works up to ~100K writes/sec
- For higher throughput + exact: use database row lock with optimistic concurrency (CAS version field)
- Pattern: `UPDATE inventory SET qty=qty-1, version=version+1 WHERE id=X AND qty>0 AND version=V`
- True exact at scale: use single-partition Kafka + single aggregator + synchronous flush (throughput bounded by DB write speed ~10K/sec)

### "getTopK must reflect real-time (< 1s lag)"
- Current: CMS updated per event; Top-K flushed to Redis every 30s
- Fix: reduce flush interval to 1s; serve Top-K from aggregator's in-memory heap directly
- Trade-off: more Redis writes; coordination between aggregators more frequent
- Alternative: stream Top-K via Server-Sent Events (SSE) — push heap updates to clients directly

### "Need Top-K per category (trending music vs trending news)"
- Maintain separate CMS + heap per category
- Kafka events tagged with category → route to category-specific aggregator partition
- Redis: `trending:{category}:{window}` sorted set per category
- Memory: each CMS = d × w × 8 bytes = 5 × 2000 × 8 = 80 KB per category per window
  100 categories × 3 windows = 300 × 80 KB = 24 MB — negligible

### "Counter must support decrement (unlike, downvote)"
- Sharded INCR/DECR both work in Redis (negative values fine)
- Write path: produce event with delta=-1 → aggregator applies negative batch sum → INCRBY shard -N
- Count-Min Sketch: does NOT support decrements (only min of non-negative counts)
- For Top-K with decrements: use Space-Saving algorithm (supports negative updates) or full sorted set

### "Constraint: reduce Redis memory 10× (cost)"
- Current: 5B shard keys × 60B = 300 GB Redis
- Fix: reduce shards from 10 → 2 for most counters (long-tail counters don't need 10 shards)
- Hot counter detection: monitor write rate → only scale to 10 shards if rate > 10K/sec
- Compression: store counter_id as hash (8B) not full string (20B) → 2.5× savings
- Evict cold counters from Redis: if no increment in 24h → persist to Cassandra, remove from Redis; reload on next access
