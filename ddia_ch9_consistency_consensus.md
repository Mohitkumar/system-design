# DDIA Chapter 9: Consistency and Consensus

## Theory

### Consistency Spectrum (weakest → strongest)
```
Eventual Consistency → Causal Consistency → Linearizability
```

- Stronger guarantees = worse performance, easier correctness reasoning
- Most replicated DBs provide only **eventual consistency** by default

---

## Linearizability

> System appears to have **one copy of data**, all operations are atomic.

- Once any read returns a new value, **all subsequent reads** (by any client) must also return that value
- No concurrent reads returning old value after a new value was seen
- Testable: record request/response timings, verify total sequential order

### Linearizability vs. Serializability
| Property | Type | Guarantee |
|----------|------|-----------|
| Serializability | Isolation (txs) | Txs behave as if executed serially |
| Linearizability | Recency (reads/writes) | Reads always see latest write |

- 2PL and actual serial execution → linearizable
- SSI → **not linearizable** by design

### When Linearizability Required
- Leader election (lock must be held by one node)
- Uniqueness constraints (usernames, primary keys)
- Cross-channel timing dependencies (file uploaded → job triggered)
- Some constraints (foreign keys) do **not** require it

### Implementation Support
| Replication Type | Linearizable? |
|-----------------|--------------|
| Single-leader (sync reads from leader) | Yes |
| Consensus algorithms | Yes |
| Multi-leader | No |
| Leaderless (Dynamo-style) | No (not recommended) |

### CAP Theorem
- Systems **not** requiring linearizability → more tolerant of network partitions
- Linearizability = slow even without network faults (coordination cost)

---

## Ordering Guarantees

### Causality vs. Total Order
| Model | Definition | Example |
|-------|-----------|---------|
| Partial order (causality) | Concurrent ops can be in any order | Causal consistency |
| Total order (linearizability) | Any two ops can be compared | Linearizable systems |

- **Causal consistency** = strongest model that doesn't slow down on network failures
- Linearizability **implies** causal consistency (stronger superset)

### Tracking Causality
- Version vectors: accurate but impractical at scale
- **Sequence numbers / logical clocks**: compact, total order
- **Lamport timestamps**: `(counter, nodeID)` — each node tracks max seen, includes in every request
  - Provides total order but **can't distinguish concurrent from causal**

### Total Order Broadcast
- Protocol guaranteeing: no message loss + totally ordered delivery to all nodes
- Used for: DB replication, serializable transactions, message logs, fencing tokens
- Equivalent to consensus

---

## Distributed Transactions and Consensus

### Why Consensus?
- Leader election: all nodes must agree who the leader is
- Atomic commit: all nodes must agree to commit or abort

### Two-Phase Commit (2PC)
```
Phase 1 (Prepare):  Coordinator → "Are you ready to commit?"
                    All participants → "Yes" / "No"

Phase 2 (Commit):   If all yes → Coordinator → "Commit"
                    Else       → Coordinator → "Abort"
```
- Two irrevocable points: participant votes yes, coordinator decides
- **If coordinator crashes after prepare but before commit** → participants blocked waiting
- 3PC theoretically solves this, not practical (assumes synchronous network)

### 2PC Problems
- Coordinator failure → participants hold locks → rest of app blocked
- ~10x slower than single-node transactions
- Manual intervention or heuristic decisions to unblock stuck txs

### Distributed Tx Types
| Type | Description |
|------|-------------|
| Internal | Same database software across nodes — can optimize |
| Heterogeneous (XA) | Different systems, standard protocol — limited optimization |

---

## Consensus Algorithms

### Properties (formalized)
1. **Agreement**: all nodes decide the same value
2. **Integrity**: no node decides twice
3. **Validity**: decided value was proposed by some node
4. **Termination**: every non-crashed node eventually decides (requires majority alive)

### Known Algorithms
- **Paxos** — foundational, complex
- **Raft** — designed for understandability, widely used
- **Zab** — used in ZooKeeper
- **Viewstamped Replication** — early academic work

All use epochs: leader uniqueness guaranteed only within one epoch. Two rounds of voting per epoch: elect leader → leader proposes → nodes vote.

### Consensus Limitations
- Requires strict majority → sensitive to node failures
- Assumes fixed set of voting nodes (dynamic membership is hard)
- Relies on timeouts to detect failed nodes → false positives under network issues

---

## ZooKeeper / etcd (Coordination Services)

Built on consensus, provide:
- **Linearizable atomic operations** (CAS, compare-and-set)
- **Total ordering of operations** with fencing tokens
- **Failure detection** via sessions + heartbeats
- **Change notifications** (watches)
- **Service discovery** and **work allocation**

Usually used **indirectly** via libraries (Kafka leader election, HBase master election, etc.)

---

## System Design Implications
- Default to causal consistency where possible — avoids linearizability cost
- Use ZooKeeper/etcd for leader election, distributed locks, service discovery — don't implement from scratch
- Avoid 2PC in hot paths — design around it using idempotency + async coordination
- SSI for high-throughput serializable workloads; 2PL only when contention is low
- Every consensus system requires majority quorum — plan node count (odd numbers: 3, 5, 7)
