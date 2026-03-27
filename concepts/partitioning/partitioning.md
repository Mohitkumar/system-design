# Partitioning (Sharding)

## Goal
- Each record belongs to exactly one partition
- Spread data and load **evenly** — avoid hot spots (skewed partitions)
- Partitioning + replication are independent schemes used together

---

## Partitioning Strategies

### By Key Range
- Sort keys, assign contiguous key boundaries per partition
- Supports efficient **range queries**
- Risk: hot spots when access is sequential (e.g., all writes go to today's time-series partition)
- Requires continuous boundary adjustment as data grows

### By Hash Key
- Hash the key (MurmurHash, MD5), partition by hash range
- More uniform distribution, avoids hot spots
- **Loses range query ability** — scatter/gather required
- Compound key trick: hash first column, range on rest → e.g., `hash(user_id) + timestamp` supports "all events for user X in time range"

### Hot Spot Mitigation
- Append random bytes to hot key → scatter writes across partitions
- Trade-off: reads must query all partitions and merge results
- Application must track and manage the suffix

---

## Secondary Index Partitioning

| Type | How | Read | Write |
|------|-----|------|-------|
| **Local (document-based)** | Each partition indexes its own data only | Scatter/gather all partitions | Fast — single partition |
| **Global (term-based)** | Index partitioned by term value | Single partition — efficient | Slow — must update multiple partitions; usually **async** |

### Local Index (Document-Based Partitioning)

Each partition maintains its own independent index covering only the data in that partition.

```
Partition 0: docs {0,1,2}   → local index: color:red → [0,2], color:blue → [1]
Partition 1: docs {3,4,5}   → local index: color:red → [3],   color:blue → [4,5]

Query "color:red" → must ask ALL partitions → merge [0,2] + [3] = [0,2,3]
```

- **Write**: only the partition owning the document is updated → single partition write, no coordination
- **Read**: scatter/gather — query all N partitions in parallel, merge results → high read fan-out
- **Failure**: if one partition is slow/down, the whole query waits or returns partial results
- **Used by**: Elasticsearch, MongoDB, Cassandra (secondary index per node)

### Global Index (Term-Based Partitioning)

A single global index is itself partitioned — not by the primary key, but by the **index term** (the value being indexed).

```
Term partition 0 (a–r): color:blue → [1,4,5],  color:red → [0,2,3]
Term partition 1 (s–z): color:yellow → [6,8],  color:white → [7]

Query "color:red" → go to term partition 0 only → get [0,2,3]  ← 1 partition read
```

- **Read**: efficient — route directly to the term's partition; no scatter/gather
- **Write**: expensive — one document write may update terms across multiple index partitions (e.g., inserting a doc with `color=red, brand=Nike` updates two different term partitions)
- **Consistency**: writes to multiple index partitions are usually **async** → index may lag behind primary data by seconds
- **Used by**: DynamoDB GSI (async), Elasticsearch cross-shard terms, Solr SolrCloud

### Partition by Term — How Terms Are Assigned to Partitions

Two strategies for deciding which partition owns a term:

| Strategy | How | Tradeoff |
|----------|-----|----------|
| **Hash of term** | `partition = hash(term) % N` | Uniform distribution; range queries on terms impossible |
| **Term range** | Alphabetical ranges per partition (a–r on p0, s–z on p1) | Range queries on index terms work; risk of hot terms (e.g., `status:active` gets all traffic) |

**Example — DynamoDB GSI:**
- GSI is a global index partitioned by the GSI partition key (term)
- Write to base table → async replication to GSI partition owning that term value
- Read from GSI → single partition scan (if GSI PK supplied) or full GSI scan
- GSI has its own WCU/RCU budget — if GSI partition is throttled, base table writes succeed but GSI lags

---

## Rebalancing Partitions

Requirements: fair load after rebalance, DB stays available during rebalance, minimal data movement.

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| Fixed partitions | More partitions than nodes; new node steals some | Simple ops | Partition size can grow too large/small |
| Dynamic partitioning | Split when too large, merge when too small | Adapts to data volume | Cold start (starts with 1 partition) |
| Per-node proportion | Fixed partitions per node; new node splits and takes half | Balances with node count | Random splits may be unfair |

**Always prefer human-in-the-loop** for rebalancing — avoid fully automated (risks cascading moves during normal load spikes).

---

## Request Routing (Service Discovery)

Problem: given key K, which node handles the request?

| Approach | Where routing lives |
|----------|-------------------|
| Each node knows all | Node redirects request to correct peer |
| Routing tier | Dedicated load balancer layer |
| Client-side | Client tracks partition map directly |

### Coordination Services
- **ZooKeeper**: keeps cluster metadata + partition assignments; nodes subscribe to changes
- **Gossip protocol** (Cassandra, Riak): nodes propagate metadata; no external coordination service — clients may contact any node

---

## Consistent Hashing

Used in leaderless/DHT systems (Dynamo, Cassandra).

```
Hash ring: 0 ─────────────────────────── 2^32
Nodes placed at positions on ring
Key → hash → walk clockwise → first node = owner
```

- Adding/removing node: only adjacent keys move — minimizes data movement
- **Virtual nodes (vnodes)**: each physical node owns multiple positions → better load distribution + easier rebalancing

---

## Summary

| Use Case | Partitioning Strategy |
|----------|-----------------------|
| Uniform KV access | Hash partitioning |
| Range queries / time-series | Key-range partitioning |
| Secondary index reads | Global (term-based) index |
| Secondary index writes | Local (document-based) index |
| Dynamic data growth | Dynamic partitioning |
| Large stable clusters | Fixed or per-node partitions |
