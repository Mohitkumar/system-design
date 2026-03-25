# Distributed Cache

**Popular implementations:** Redis, Memcached, Hazelcast

---

## Writing Policies

| Policy | How | Tradeoff |
|---|---|---|
| **Write-through** | Write to cache + DB synchronously | Consistent, higher write latency |
| **Write-back** | Write to cache first, flush to DB async | Low latency, risk of data loss on crash |
| **Write-around** | Write directly to DB, skip cache | Avoids caching rarely-re-read data; first read is always a miss |

---

## Eviction Policies

| Policy | Evicts | Best for |
|---|---|---|
| **LRU** | Least recently used | General-purpose; works well when recency predicts future access |
| **LFU** | Least frequently used | Stable hot-key workloads (e.g. product catalog) |
| **MRU** | Most recently used | Cyclic scans where the just-used item won't be needed again soon |
| **TTL-based** | Expired entries | Session data, rate-limit counters, anything with a natural expiry |
| **Random** | Random entry | Simple; surprisingly competitive under uniform access patterns |

---

## Cache Invalidation Strategies

| Strategy | How |
|---|---|
| **TTL expiry** | Background thread (lazy or active) checks timestamps; evicts on expiry |
| **Lazy / read-time** | On read, check TTL — if expired, delete and fetch fresh from DB |
| **Write-invalidate** | On DB write, explicitly delete or update the cache entry |
| **Event-driven** | DB change event (CDC / Debezium) triggers cache invalidation |

**Cache stampede problem:** When a hot key expires, many requests simultaneously miss and hammer the DB. Fix with:
- **Mutex/lock on miss** — only one request fetches; others wait.
- **Probabilistic early expiration** — refresh slightly before TTL expires under high load.
- **Background refresh** — serve stale while asynchronously fetching fresh data.

---

## Internal Storage Mechanism

**LRU cache internals:** `HashMap` (O(1) key lookup) + `Doubly Linked List` (O(1) move-to-front / evict-tail).

```
HashMap: key → node pointer
LinkedList: [MRU] ← node ↔ node ↔ ... ↔ node → [LRU]

On get(key):  find node via map, move to head.
On put(key):  insert at head; if over capacity, evict tail node.
```

---

## Sharding Strategies

| Strategy | How | Tradeoff |
|---|---|---|
| **Dedicated per service** | Each service owns its cache namespace | Simple isolation; wasted capacity if services are uneven |
| **Modulo hashing** | `server = hash(key) % N` | Simple; adding/removing a node remaps ~all keys (thundering herd) |
| **Consistent hashing** | Keys and nodes on a ring; key maps to nearest clockwise node | Adding/removing a node remaps only `1/N` of keys |
| **Consistent hashing + virtual nodes** | Each physical node owns multiple points on the ring | Better load balance; standard in Redis Cluster, Cassandra |

---

## Replication & Availability

| Pattern | How | Used by |
|---|---|---|
| **Primary–Replica** | Writes go to primary; replicas serve reads | Redis Sentinel |
| **Multi-primary** | Any node accepts writes; conflict resolution needed | Redis Enterprise Active-Active (CRDTs) |
| **No replication** | Cache is ephemeral; DB is source of truth | Memcached |

**Redis Cluster:** 16384 hash slots split across nodes. Each slot is a consistent-hash bucket. Nodes gossip membership; clients can be redirected with `MOVED`/`ASK`.

---

## Cache Aside vs Read-Through vs Refresh-Ahead

| Pattern | Who manages cache | Flow |
|---|---|---|
| **Cache aside** (lazy loading) | Application | App checks cache → miss → app fetches DB → app populates cache |
| **Read-through** | Cache library/proxy | App asks cache → miss → cache fetches DB → returns to app |
| **Refresh-ahead** | Cache | Cache predicts upcoming expiry and pre-fetches before TTL hits |

Cache aside is the most common pattern (used by most Redis integrations). It gives the app full control but couples it to cache logic.

---

## Hot Keys & Large Keys

- **Hot key:** A single key gets a disproportionate share of traffic. Fix: local in-process cache (L1) in front of Redis, or shard the key into `key#0 .. key#N`.
- **Large value:** Storing a 10MB blob in Redis blocks the single-threaded event loop. Fix: store large blobs in S3/blob store; cache only the metadata or a pointer.

---

---

# Distributed Cache — System Design

---

## Functional Requirements

- `put(key, value, ttl?)` — store a value with optional expiry
- `get(key)` → value or null — retrieve a value
- `delete(key)` — explicit invalidation
- Support TTL-based auto-expiry

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Read latency | < 5ms p99 |
| Write latency | < 10ms p99 |
| Availability | 99.99% |
| Consistency | Eventual (cache can be slightly stale) |
| Durability | Best-effort (cache is a performance layer, not source of truth) |
| Scalability | Horizontal — add nodes without downtime |

---

## Capacity Estimation

**Assumptions:** 100M DAU, avg 10 reads/user/day = 1B reads/day ≈ **~12K reads/sec**. Write ratio 1:10 → ~1.2K writes/sec.

**Storage:**
- Avg key: 100 bytes, avg value: 1 KB → ~1.1 KB per entry
- **80/20 rule:** 20% of keys serve 80% of traffic → cache only that hot 20%
- If total dataset = 500 GB → cache = 500 GB × 20% = **100 GB**
- Avg entry size 1.1 KB → ~90M entries

**Nodes:**
- Each node holds 32 GB RAM (leaving headroom for OS + overhead)
- 100 GB / 32 GB ≈ **4 data nodes** (add replicas on top)

---

## High-Level Architecture

```
Clients
   │
   ▼
[ Cache Client Library ]   ← consistent hashing, connection pooling
   │             │
   ▼             ▼
[Shard 0]    [Shard 1]  ...  [Shard N]     ← each shard = primary + replicas
   │
[Primary]  ──async repl──►  [Replica 1]
                        ──async repl──►  [Replica 2]
```

---

## Sharding

**Consistent hashing with virtual nodes** (standard approach):

- Hash ring of 2^32 positions.
- Each physical node owns `V` virtual nodes (e.g. V=150) spread around the ring.
- `key → hash(key) → walk ring clockwise → nearest virtual node → physical node`
- Adding/removing a node remaps only `1/N` of keys; virtual nodes ensure even load.

**Shard map storage:** A central config service (ZooKeeper / etcd) stores the ring topology. Clients cache it locally and refresh on `MOVED` errors or heartbeat.

---

## Replication

Each shard runs **1 primary + 2 replicas** (configurable).

### Sync vs Async Replication

| | Sync | Async |
|---|---|---|
| **How** | Primary waits for ACK from ≥1 replica before returning success | Primary returns immediately; replicates in background |
| **Durability** | Strong — no data loss on primary crash | Potential loss of last N writes on crash |
| **Latency** | Higher (adds replica RTT to write path) | Lower |
| **Availability** | Lower — write blocked if replica is slow/down | Higher |
| **When to use** | Financial data, session tokens | General cache (slight staleness acceptable) |

**Recommended:** async replication for cache (cache is not source of truth). Use **semi-sync** (wait for 1 of 2 replicas) if durability matters.

---

## Persistence

Cache is primarily in-memory, but optional persistence protects against cold-start thundering herd after a restart.

| Mechanism | How | Tradeoff |
|---|---|---|
| **No persistence** | Pure in-memory | Fastest; full cold-start on restart |
| **Periodic snapshot (RDB)** | Fork process, dump full memory to disk every N mins | Fast restart load; can lose last N mins of writes |
| **Append-only log (AOF)** | Every write appended to log file; replay on restart | Near-zero data loss; slower restart (replay all ops) |
| **Hybrid** | RDB for base + AOF for delta since last snapshot | Balanced — used by Redis default config |

For pure cache use cases (DB is source of truth), **no persistence** or **RDB-only** is fine.

---

## Availability

**Per-shard:** Primary + 2 replicas in different AZs. If primary dies, a replica is promoted.

**Failure detection:** Nodes gossip heartbeats every 100ms. After `N` missed heartbeats (~500ms), node is marked suspect → dead. Configurable to avoid false positives under GC pauses.

**Shard Failover (primary crash):**

```
1. Replica detects primary missed heartbeats → marks it DOWN
2. Replicas hold an election (Raft or Sentinel-style)
   - Candidate with most up-to-date replication offset wins
3. Winner promotes itself to primary
4. Other replicas re-point to new primary
5. Config service updates shard map
6. Clients receive MOVED / reconnect
```

Failover target: **< 30 seconds** for automated promotion.

**Split-brain prevention:** Require majority quorum (2 of 3 nodes) to elect a new primary. A partitioned replica that can't reach majority stays read-only.

---

## Serialization

Keys are always strings (UTF-8). Values need a wire format:

| Format | Size | Speed | Schema | Use |
|---|---|---|---|---|
| **JSON** | Large | Slow | None | Debugging, simple objects |
| **MessagePack** | Small | Fast | None | Drop-in binary JSON |
| **Protobuf** | Smallest | Fastest | Required | High-throughput, typed data |
| **raw bytes** | — | — | App-defined | Images, blobs |

**Recommendation:** Protobuf for structured data; raw bytes for media. Always version your schema (include a version byte prefix) so deserialization survives rolling deploys.

---

## Summary Cheatsheet

| Design Decision | Choice | Reason |
|---|---|---|
| Sharding | Consistent hashing + vnodes | Minimal remapping on scale-out |
| Replication | 1 primary + 2 replicas, async | Low write latency; eventual consistency acceptable |
| Failover | Quorum election (Raft-style) | Prevents split-brain |
| Eviction | LRU + TTL | General-purpose; TTL handles stale data |
| Write policy | Cache-aside | App controls freshness |
| Persistence | RDB snapshots (optional) | Warm restart; not source of truth |
| Serialization | Protobuf | Compact, fast, versioned |
| Invalidation | TTL + write-invalidate on DB update | Balances simplicity and freshness |

---

## Quick Comparison: Redis vs Memcached

| | Redis | Memcached |
|---|---|---|
| Data structures | Strings, hashes, lists, sets, sorted sets, streams | String only |
| Persistence | RDB snapshots + AOF log | None |
| Replication | Built-in | None (client-side sharding only) |
| Clustering | Redis Cluster (built-in) | Client-side consistent hashing |
| Lua scripting | Yes | No |
| Best for | Feature-rich caching, pub/sub, leaderboards, queues | Pure high-throughput key-value cache |
