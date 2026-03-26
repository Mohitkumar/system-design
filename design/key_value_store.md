# Distributed Key-Value Store

**Examples:** DynamoDB, Cassandra, Riak, Redis Cluster, etcd, TiKV

---

## Theory

### What is a KV Store?

Simplest possible data model: `put(key, value)`, `get(key)`, `delete(key)`. No schema, no joins. Scales horizontally because each key is independent.

### Why Distributed?

Single node: limited RAM/disk, single point of failure.
Distributed: data partitioned across nodes, replicated for durability and availability.

### The Core Tension

Every design decision in a distributed KV store is a tradeoff between:
- **Consistency** — all nodes see the same data at the same time
- **Availability** — every request gets a response
- **Partition tolerance** — system works despite network splits

---

## CAP Theorem

Network partitions are unavoidable → you must choose between C and A during a partition.

| Type | Behavior during partition | Examples |
|---|---|---|
| **CP** | Reject writes to preserve consistency | etcd, ZooKeeper, HBase |
| **AP** | Accept writes, resolve conflicts later | DynamoDB, Cassandra, Riak |
| **CA** | Impossible in distributed systems | — |

---

## PACELC

CAP only covers partition scenarios. **PACELC** extends it to normal operation:

> **If Partition** → choose between **A**vailability and **C**onsistency
> **Else** (no partition) → choose between **L**atency and **C**onsistency

```
                  ┌─ Partition? ─────┐
                  │ YES              │ NO
                  ▼                  ▼
           A  vs  C           L  vs  C
```

| System | Partition | Else | Notes |
|---|---|---|---|
| DynamoDB | A | L | Eventual consistency by default; strong consistency optional at higher latency |
| Cassandra | A | L | Tunable consistency (ONE/QUORUM/ALL) |
| etcd / ZooKeeper | C | C | Strong consistency always; higher latency |
| Riak | A | L | Eventual; CRDTs for auto-merge |
| Spanner | C | C | Uses TrueTime to provide external consistency globally |

---

## Partitioning Strategies

How to decide which node owns a given key.

### 1. Modulo Hashing

```
node = hash(key) % N
```

- Simple.
- **Problem:** Adding/removing a node remaps `~all` keys → massive data movement.

### 2. Consistent Hashing

Keys and nodes placed on a virtual ring (0 to 2^32). A key is owned by the first node clockwise from its hash position.

```
Ring:  ... [Node A at 100] ... [Node B at 200] ... [Node C at 300] ...
Key hash = 150 → owned by Node B
Key hash = 250 → owned by Node C
```

Adding/removing a node remaps only `1/N` of keys.

### 3. Consistent Hashing + Virtual Nodes (vnodes)

Each physical node owns `V` virtual positions on the ring (e.g. V=150).

- Better load balance — physical nodes each get a proportional slice regardless of ring position luck.
- Adding a node steals small ranges from many existing nodes → smooth rebalancing.
- **Used by:** Cassandra, DynamoDB, Riak.

### 4. Range Partitioning

Key space sorted; each node owns a contiguous key range.

```
Node A: keys  a–m
Node B: keys  n–z
```

- Efficient range scans.
- **Risk:** Hot partitions if key distribution is skewed (e.g. all writes to "a" prefix).
- **Used by:** HBase, Bigtable, TiKV.

---

## Replication Strategies

### Leader-Based (Primary-Replica)

All writes go to the leader; leader replicates to followers.

```
Client → Leader → Follower 1
                → Follower 2
```

- **Sync replication:** leader waits for all followers → strong consistency, higher latency.
- **Async replication:** leader returns immediately → low latency, risk of data loss on leader crash.
- **Semi-sync:** wait for ≥1 follower → balance. Used by MySQL, Kafka.

#### Raft (Leader Election & Log Replication)

Raft is the consensus algorithm that makes leader-based replication safe — it answers: *who is the leader, and how do we guarantee followers don't diverge?*

**Roles:**
- **Leader** — handles all writes; sends log entries to followers.
- **Follower** — passive; replicates entries from leader.
- **Candidate** — temporary role during election.

**Leader Election:**

```
1. All nodes start as Followers with a random election timeout (150–300ms).
2. If no heartbeat from leader before timeout → become Candidate.
3. Candidate increments its term, votes for itself, requests votes from peers.
4. Node grants vote if:
     a. Candidate's term ≥ own term
     b. Candidate's log is at least as up-to-date as own log
     c. Haven't voted in this term yet
5. Candidate wins majority (⌊N/2⌋ + 1 votes) → becomes Leader.
6. Leader sends heartbeats every ~50ms to suppress new elections.
```

- Random timeouts prevent split votes. If a split vote occurs, nodes wait for a new timeout and retry.
- **Term** — monotonic counter, acts as a logical clock. Any node seeing a higher term steps down to Follower.

**Log Replication:**

```
Client write → Leader
  1. Leader appends entry to its local log (uncommitted)
  2. Leader sends AppendEntries RPC to all followers in parallel
  3. Followers append to their logs, reply ok
  4. Once majority ack → Leader marks entry COMMITTED
  5. Leader applies to state machine, returns success to client
  6. Next heartbeat / AppendEntries tells followers the commit index
  7. Followers apply committed entries to their state machines
```

**Safety guarantee:** A committed entry is present on a majority of nodes. A new leader must have all committed entries (ensured by vote restriction in step 4b above). Therefore committed data is never lost on leader change.

**Log Consistency:**
- Each AppendEntries includes `(prevLogIndex, prevLogTerm)`.
- Follower rejects if its log doesn't match → leader walks back and retries from the divergence point.
- Ensures all committed logs are identical across nodes.

**Used by:** etcd, CockroachDB, TiKV, Kafka KRaft, Consul.

### Leaderless (Dynamo-style)

Any node can accept reads and writes. Coordinator routes to `N` replica nodes.

```
Client → Coordinator
              ├─ write to Replica 1
              ├─ write to Replica 2
              └─ write to Replica 3
         return success after W acknowledgements
```

**Quorum parameters:**
- `N` = total replicas per key
- `W` = write quorum (must ack before success)
- `R` = read quorum (must respond before return)

| Condition | Guarantee |
|---|---|
| `W + R > N` | Strong consistency (reads always see latest write) |
| `W + R ≤ N` | Eventual consistency (faster, possible stale reads) |
| `W = N, R = 1` | Fast reads, slow writes |
| `W = 1, R = N` | Fast writes, slow reads |
| Typical: `N=3, W=2, R=2` | Balanced — tolerates 1 node failure |

**Used by:** DynamoDB, Cassandra, Riak.

### Multi-Leader

Multiple nodes accept writes; sync in background. Used for geo-distributed deployments.

- **Pros:** writes to local region → very low latency.
- **Cons:** write conflicts are common → need a conflict resolution strategy.
- **Used by:** CockroachDB (multi-region), DynamoDB global tables.

---

## Consistency Models

From strongest to weakest:

| Model | Guarantee | Latency |
|---|---|---|
| **Linearizability** | Every read sees the most recent write; operations appear instantaneous | Highest |
| **Sequential consistency** | All nodes see operations in the same order (not necessarily real-time) | High |
| **Causal consistency** | Causally related operations seen in order; concurrent writes may diverge | Medium |
| **Eventual consistency** | All replicas converge eventually if writes stop | Lowest |

**Tunable consistency (Cassandra):**

```
ONE   → fastest, weakest (1 node must ack)
QUORUM → balanced (majority must ack)
ALL   → slowest, strongest (all replicas must ack)
LOCAL_QUORUM → quorum within local datacenter only
```

---

## Versioning

When the same key is written concurrently on different nodes, you need to track which version is newer.

### Timestamp (Wall Clock)

Attach a timestamp to every write. Latest timestamp wins.

- **Problem:** Clocks across nodes are not perfectly synchronized (NTP drift ±100ms). Can silently discard a newer write.

### Lamport Clock

Logical counter. Every write increments it. Tie-break by node ID.

- **Problem:** Total order, but causally blind — doesn't distinguish concurrent writes from sequential ones.

### Vector Clock

Each node maintains a counter for every other node: `{A:1, B:2, C:0}`.

```
Write on Node A:  version = {A:1, B:0, C:0}
Write on Node B:  version = {A:0, B:1, C:0}

Compare:
  {A:1,B:0} vs {A:0,B:1} → neither dominates → CONFLICT
  {A:1,B:1} vs {A:1,B:0} → first dominates   → NO CONFLICT (second is ancestor)
```

- Detects true conflicts vs causal ordering.
- **Problem:** Vector grows with number of nodes. Riak caps size and truncates old entries.

### Hybrid Logical Clocks (HLC)

Combines physical time (NTP) with a logical counter:
`hlc = (physicalMs, logicalCounter)`

- Stays close to real time (useful for debugging, TTL).
- Handles NTP corrections without going backward.
- **Used by:** CockroachDB, YugabyteDB.

---

## Conflict Detection & Resolution

### Detection

| Method | How | Used by |
|---|---|---|
| Vector clocks | Compare version vectors; divergent = conflict | Riak, DynamoDB (internally) |
| Merkle trees | Hash subtrees of key ranges; divergent subtree = replica out of sync | DynamoDB, Cassandra (anti-entropy) |
| Read-repair | On read, coordinator fetches all replicas; inconsistency = stale replica | Cassandra, Riak |

### Resolution

| Strategy | How | Data loss? | Used by |
|---|---|---|---|
| **Last Write Wins (LWW)** | Highest timestamp/version wins | Yes — concurrent write lost | Cassandra default, DynamoDB |
| **First Write Wins** | Lowest version wins; later rejected | Yes | Locks, leader election |
| **Client-side merge** | Return all conflicting versions ("siblings") to client; app merges | No | Riak, DynamoDB (conditional writes) |
| **CRDTs** | Special data types that merge automatically (counters, sets, registers) | No | Riak, Redis Enterprise, Cosmos DB |
| **Operational Transform** | Merge concurrent edits by transforming operations against each other | No | Google Docs, collaborative editors |

### CRDT Examples

| Type | Operation | Merge Rule |
|---|---|---|
| G-Counter | increment only | sum of all per-node counters |
| PN-Counter | increment + decrement | sum(increments) − sum(decrements) per node |
| G-Set | add only | union |
| OR-Set | add + remove | remove wins only if it observed the add |
| LWW-Register | set value | highest timestamp wins |

---

# KV Store — System Design

---

## Functional Requirements

- `put(key, value, ttl?)` → ok
- `get(key)` → value | not_found
- `delete(key)` → ok
- Configurable consistency per request (eventual / strong)

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | 99.99% |
| Durability | No data loss after `put` returns |
| Latency | get/put < 10ms p99 (eventual), < 50ms p99 (strong) |
| Scalability | Petabyte-scale, 100K ops/sec per node |
| Fault tolerance | Survive up to `f` node failures (with `2f+1` replicas) |

---

## Capacity Estimation

**Assumptions:** 500M keys, avg 10 KB value, 1M reads/sec, 100K writes/sec.

- **80/20 rule:** 80% reads in 20% of the day → peak reads = (1M × 0.8) / (0.2 × 86,400) ≈ **~46K reads/sec**; design for 3× = **~150K reads/sec**
- **Storage:** 500M × 10 KB = **5 TB** raw; 3× replication = **15 TB**
- **Nodes:** 15 TB / 2 TB per node (leaving headroom) = **~8 nodes** storage; more for throughput

**IOPS (LSM-tree storage engine):**
- Write IOPS: 1 sequential write per op (WAL append) → 100k writes/sec × 1 = **100k write IOPS** (sequential)
- Read IOPS: bloom filter + sparse index → ~2–4 random reads per cache miss
  - Assume 10% cache miss rate: 150k reads/sec × 10% × 3 IOs = **45k random read IOPS**
- Total: ~145k IOPS; NVMe SSD = 500k IOPS → **1 NVMe per storage node sufficient**
- Compaction burst: up to 2× write IOPS during compaction; schedule off-peak or rate-limit

**Server count:**
| Layer | Count | Sizing | Reason |
|-------|-------|--------|--------|
| Coordinator / API | 5 | 16 cores, 32 GB | 150k req/sec ÷ 30k per server |
| Storage nodes | 12–15 | 16 cores, 64 GB RAM, 2TB NVMe | 15 TB ÷ 2TB + 3× replication spread |
| ZooKeeper / etcd | 3–5 | 8 cores, 16 GB | Quorum for membership/config |

**Memory per storage node (RocksDB block cache):**
- Cache 10% of hot data: 500 GB ÷ 12 nodes = 42 GB → set block cache to **32–48 GB per node**
- Bloom filter overhead: ~10 bits/key × 42M keys/node ÷ 8 = ~52 MB per node (negligible)

---

## High-Level Architecture

```
Client SDK
  │ routes request via consistent hash ring
  ▼
[ Coordinator Node ]  ← any node can be coordinator (leaderless)
  │
  ├─ put(key, value) → hash(key) → find N replica nodes
  │     ├─ write to Replica 1 ──┐
  │     ├─ write to Replica 2   ├─ wait for W acks → return ok
  │     └─ write to Replica 3 ──┘
  │
  └─ get(key) → find N replica nodes
        ├─ read from Replica 1 ──┐
        ├─ read from Replica 2   ├─ wait for R responses → reconcile → return
        └─ read from Replica 3 ──┘

[ Gossip Protocol ]  ← nodes exchange membership, failure detection, ring topology
[ Anti-entropy ]     ← background Merkle tree sync to repair stale replicas
[ Hinted Handoff ]   ← if replica is down, coordinator stores write temporarily and replays when it recovers
```

---

## Low-Level Design

### Node Storage Engine

Each node stores its key range using an **LSM-tree** (Log-Structured Merge-tree):

```
Write path:
  1. Write to WAL (crash recovery)
  2. Write to in-memory MemTable (sorted map)
  3. When MemTable full → flush to immutable SSTable on disk
  4. Background compaction merges SSTables, removes tombstones

Read path:
  1. Check MemTable
  2. Check bloom filter per SSTable (skip if key definitely absent)
  3. Binary search SSTable index → seek to block → scan
```

**Why LSM over B-tree?** Writes are sequential (append WAL + flush) → much higher write throughput. Reads are slightly slower but bloom filters reduce disk seeks.

**Used by:** RocksDB (underlying engine for TiKV, CockroachDB, DynamoDB).

### Request Routing

```
Client SDK caches the ring topology (refreshed via gossip or config service).

put(key):
  hash = murmur3(key)
  nodes = ring.getPreferenceList(hash, N=3)   // N clockwise nodes
  coordinator = nodes[0]
  coordinator.write(nodes, key, value, W=2)

get(key):
  hash = murmur3(key)
  nodes = ring.getPreferenceList(hash, N=3)
  coordinator = nodes[0]
  coordinator.read(nodes, key, R=2)
```

### Versioning Flow (Vector Clock)

```
Initial write (Node A):
  key="cart", value=[apple], clock={A:1, B:0, C:0}

Concurrent write (Node B, didn't see A's write):
  key="cart", value=[banana], clock={A:0, B:1, C:0}

On read (R=2), coordinator gets both versions:
  version 1: {A:1,B:0} — neither dominates
  version 2: {A:0,B:1} — the other

→ return both siblings to client
→ client merges: [apple, banana], writes back with merged clock {A:1,B:1,C:0}
```

### Failure Detection (Gossip)

Each node maintains a membership list with a **heartbeat counter** per member.

```
Every 1s: node picks 2 random peers, exchanges membership list
On receive: update local list with max(local_counter, received_counter)

Node marked SUSPECT if heartbeat hasn't incremented in T seconds (e.g. 10s)
Node marked DEAD after SUSPECT for T2 seconds → trigger hinted handoff + rebalance
```

Gossip converges in `O(log N)` rounds → fast failure detection even in large clusters.

### Hinted Handoff

If a replica node is temporarily unavailable:

```
Coordinator stores the write locally with a hint:
  { key, value, version, intended_replica: Node_B }

When Node_B recovers:
  Coordinator detects recovery via gossip
  Replays hinted writes to Node_B
  Deletes local hint
```

Ensures writes are not lost due to short outages without waiting for full replication.

### Anti-Entropy (Merkle Trees)

Background process to detect and repair diverged replicas:

```
Each node builds a Merkle tree over its key range:
  Leaf = hash(value) for each key
  Parent = hash(children)

Two replicas compare root hashes:
  Same root → in sync, done
  Different root → recurse into subtrees → find diverged leaf → sync that key
```

Efficient — only diverged keys are exchanged, not the full dataset.

---

## Failure Scenarios

| Failure | Impact | Mitigation |
|---|---|---|
| Node crash | Its key ranges temporarily under-replicated | Hinted handoff + re-replication to new node |
| Network partition | Two sides accept writes independently | Vector clocks detect conflict; resolved on reconnect |
| Slow node | Reads/writes time out on that replica | Coordinator skips it after timeout; reads from other R replicas |
| Split-brain (leaderless) | Both partitions accept writes | Vector clocks surface conflict; CRDTs auto-resolve; LWW as fallback |
| Disk failure | Data lost on that node | Replica catches up via anti-entropy from peers |
| Hot key | One node overwhelmed | Shard hot key; local read cache; read from any replica |

---

## Summary Cheatsheet

| Design Decision | Choice | Reason |
|---|---|---|
| Partitioning | Consistent hashing + vnodes | Minimal remapping on scale-out; even balance |
| Replication | Leaderless, N=3 W=2 R=2 | No single point of failure; tunable consistency |
| Consistency | Eventual default; strong via W+R>N | Latency-availability tradeoff per use case |
| Versioning | Vector clocks | Detect true conflicts vs causal order |
| Conflict resolution | Client merge / CRDTs / LWW | Depends on data type; CRDTs preferred |
| Storage engine | LSM-tree (RocksDB) | High write throughput; compaction handles deletes |
| Failure detection | Gossip protocol | Decentralized; scales to thousands of nodes |
| Stale replica repair | Hinted handoff + Merkle anti-entropy | Short outage: handoff; long drift: Merkle sync |
| Clock | Hybrid Logical Clocks | Close to real time; survives NTP corrections |

---

## Interview Scenarios

### "Require strong consistency for financial data"
- Switch from AP (W=2, R=2, N=3) to CP: set W=3 (quorum write to all replicas before ack)
- Or use single-leader per key range (like etcd/ZooKeeper) — writes go to leader, linearizable reads from leader
- Trade-off: write latency increases (wait for slowest replica); availability drops during minority partition
- For payments: use CP store (etcd) or add compare-and-swap (`CAS`) operations to prevent double-spend

### "Hot key — one celebrity user_id gets 1000× normal traffic"
- Local in-process cache (L1) on every app server: serve from Caffeine/Guava before hitting KV store
- Shard the key: `user_1234#0` through `user_1234#9` → write to all, read from random shard → 10× throughput
- Read from any replica (not just W replicas) for hot reads — add read-only replicas for hot keys
- Detect with `OBJECT FREQ` (LFU stats) or access log sampling; apply targeted fix per hot key

### "Large value storage — storing 50 MB video metadata blobs"
- KV store is optimized for small values (<10 KB); large values amplify compaction, GC, and network
- Store large value in S3/blob store; KV stores only a pointer (`s3://bucket/key`)
- If must store inline: use chunking — split blob into 1 MB chunks, store as `key#0`..`key#N`; reassemble on read
- Use separate LSM column family or separate cluster for large values to isolate compaction overhead

### "Traffic grows 10× — 120k writes/sec"
- Current: 5 nodes handling 12k writes/s. Need 10× → 50 nodes
- Consistent hashing with vnodes: adding nodes remaps only `1/N` keys — minimal disruption
- Add nodes gradually (5 at a time); wait for data rebalance before adding more
- If write amplification is bottleneck: tune RocksDB compaction (`level_compaction_dynamic_level_bytes=true`)

### "Node failure during write — W=2, N=3, 1 node down"
- Write can still complete with W=2 (quorum met by 2 remaining nodes)
- Hinted handoff: coordinator stores the write locally with a hint for the failed node
- When failed node recovers, coordinator replays hints → eventual consistency restored
- Monitor `PendingHints` metric; if node down >1hr, trigger Merkle anti-entropy sync on recovery

### "Need range queries — find all keys from user_100 to user_200"
- Hash-based partitioning (DynamoDB, Redis) breaks range queries — keys scatter across all partitions
- Fix: switch to range-based partitioning (like HBase, TiKV, CockroachDB) — keeps ranges on same node
- Or: maintain a secondary sorted index (GSI in DynamoDB, materialized view in Cassandra)
- Trade-off: range partitioning introduces hot spots on monotonic keys (auto-increment IDs) → use UUIDs or reversed keys

### "Constraint: must survive full datacenter failure"
- Multi-region replication: async replication to secondary region (RPO = replication lag, ~seconds)
- Active-active: both regions accept writes; conflict resolution via LWW or vector clocks
- Active-passive: primary region handles all writes; secondary is read-only standby; failover in <30s
- Quorum across regions: W=2 (one local + one remote) — strong consistency but +RTT write latency
