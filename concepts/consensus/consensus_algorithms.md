# Consensus Algorithms

## What is Consensus?

Getting several nodes to agree on a single value — even when some nodes fail or messages are lost.

### Formal Properties
1. **Agreement** — all correct nodes decide the same value
2. **Integrity** — no node decides twice; decided value was proposed by some node
3. **Validity** — if all propose V, all decide V
4. **Termination** — every non-crashed node eventually decides (requires majority alive)

### Why It Matters
- **Leader election**: all nodes must agree on one leader
- **Atomic commit**: all nodes must agree to commit or abort
- **Total order broadcast** ≡ repeated rounds of consensus

---

## Partition-Tolerant Consensus: Majority Voting

Instead of unanimous agreement (2PC), use **majority quorum**.

| Node count | Tolerated failures |
|------------|-------------------|
| 3 | 1 |
| 5 | 2 |
| 7 | 3 |

**Key properties:**
- Odd number of nodes (3, 5, 7)
- Only the majority partition stays active during a split; minority halts
- One leader per **epoch** (logical term) — uniqueness guaranteed within epoch
- Two rounds of voting: elect leader → leader proposes → nodes vote on proposal

---

## Paxos

Foundational consensus algorithm by Leslie Lamport (1998).

### Two Phases (per round)
**Phase 1 — Prepare:**
- Proposer sends `Prepare(n)` to majority of acceptors
- Acceptors reply with highest previously accepted proposal (if any)

**Phase 2 — Accept:**
- Proposer sends `Accept(n, v)` where v = own value, or highest value from phase 1
- Acceptors accept if they haven't promised to a higher n

**Critical property**: "If a proposal with value v is chosen, every higher-numbered proposal chosen has value v."

### Multi-Paxos
- Run multiple Paxos rounds efficiently with a stable leader
- Leader skips phase 1 once elected (optimization)
- Cluster membership changes add complexity

### Used In
Google Chubby, Google Spanner, Google BigTable/GFS/Megastore

---

## Raft

Designed for understandability (Ongaro & Ousterhout, 2013). Equivalent guarantees to Paxos.

### Components
- **Leader election**: followers timeout → become candidate → increment term → request votes → majority wins
- **Log replication**: leader appends entry → sends to followers → commits on majority ack → applies to state machine
- **Safety**: never commits entries from previous terms unless current-term entry also committed

### Leader Election
```
Follower (timeout) → Candidate (request votes) → Leader (majority)
                   ↑____________________________↓ (lost election → back to follower)
```

### Log Replication
```
Client → Leader (append log entry)
       → Followers (AppendEntries RPC)
       → Majority ack → commit
       → Apply to state machine → respond to client
```

### Used In
etcd, CockroachDB, TiKV, Consul

---

## ZAB (Zookeeper Atomic Broadcast)

Used in Apache ZooKeeper. Technically atomic broadcast (not pure consensus) but maintains strong consistency. Open-source equivalent to Google Chubby.

Used indirectly by: HBase, Storm, Kafka (for controller election), HDFS NameNode HA.

---

## Algorithm Comparison

| | Primary/Backup | 2PC | Paxos/Raft |
|-|---------------|-----|-----------|
| Leader | Static | Static | Dynamic election |
| Failure tolerance | Manual failover | Blocks on coordinator/node failure | Tolerates n/2-1 simultaneous failures |
| Partition tolerant | No | No | **Yes** |
| Latency | Low (async) | Tail-latency sensitive | Less tail-latency sensitive |
| CAP | CA | CA | CP |

---

## Consensus Limitations

- Requires **strict majority** to operate (n/2 + 1 nodes must be up)
- Assumes **fixed set** of voting nodes — dynamic membership is hard
- Relies on timeouts to detect failed nodes → false positives under network issues
- Sensitive to network problems — performance degrades with high latency

---

## Total Order Broadcast

Protocol guaranteeing:
1. **Reliability** — no message lost (if delivered to one node, delivered to all)
2. **Total ordering** — all nodes deliver messages in same order

### Equivalence to Consensus
- Total order broadcast = repeated rounds of consensus
- Consensus = use total order broadcast to decide one value

### Uses
- Database replication (each replica applies same log in same order)
- Serializable transactions
- Distributed locks / fencing tokens (monotonically increasing sequence)
- ZooKeeper / etcd state machine replication
