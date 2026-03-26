# DDIA Chapter 6: Partitioning (Sharding)

## Theory

### Goal
- Each record belongs to exactly one partition
- Spread data/load **evenly** — avoid hot spots (skewed partitions)
- Partitioning + replication used together (independent schemes)

---

## Partitioning Strategies

### By Key Range
- Sort keys, assign boundaries per partition
- Supports **range queries**
- Risk: hot spots if access pattern is sequential (e.g. time-series writes all go to latest partition)
- Needs continuous boundary re-adjustment

### By Hash Key
- Hash the key (MD5, MurmurHash), partition by hash range
- More uniform distribution
- **Loses range query ability** — scatter/gather required
- Supports compound keys: hash first part, range on rest (e.g. `user_id + timestamp`)

### Hot Spot Mitigation
- Append random bytes to hot key → scatter across partitions
- Trade-off: reads must query all partitions and merge

---

## Secondary Index Partitioning

| Type | How | Read | Write |
|------|-----|------|-------|
| Local (document-based) | Each partition indexes its own data | Scatter/gather all partitions | Fast (single partition) |
| Global (term-based) | Index partitioned by term | Efficient (single partition) | Slow (many partitions); usually async |

---

## Rebalancing Partitions

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| Fixed partitions | More partitions than nodes; new node steals some | Simple | Partition size can become too large/small |
| Dynamic partitioning | Split when too large, merge when too small | Adapts to data volume | Cold start (1 partition initially) |
| Per-node proportion | Fixed partitions per node; split on join | Balances with node count | Random splits may be unfair |

- **Always prefer human-in-the-loop** for rebalancing — avoid fully automated (risks cascading moves during normal load spikes)

---

## Request Routing

Problem: which node handles request for key K?

| Approach | Where routing logic lives |
|----------|--------------------------|
| Each node knows all | Node redirects to correct peer |
| Routing tier | Dedicated load balancer layer |
| Client-side | Client tracks partition map |

- Most use **ZooKeeper** to track cluster metadata + partition assignments
- Cassandra/Riak use gossip protocol (no external coordination service)

---

## System Design Implications
- Hash partition for write-heavy, uniform access (user data, events)
- Range partition for time-series, ordered scans
- Always account for secondary index cost in write-heavy systems
- Plan partition count for expected data growth (fixed-partition systems)
