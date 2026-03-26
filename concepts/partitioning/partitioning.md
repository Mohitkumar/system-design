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
