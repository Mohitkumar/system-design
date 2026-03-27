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

### Raft Term — The Core Safety Mechanism

A **term** is a monotonically increasing logical epoch. Each election starts a new term.

```
Term 1: Node A is leader
        [A]──► [B] [C]   (replicating log entries)

Term 2: A crashes. B times out → starts election → increments term to 2
        [B becomes candidate, requests votes]
        B gets vote from C → majority (2/3) → B is leader for term 2
        [B]──► [C]  (A is partitioned/dead)
```

**Every message carries a term number.** If a node sees a higher term, it immediately reverts to follower and discards its leader status.

### Raft Node Failure Scenarios

**Scenario 1: Follower crashes**
```
[Leader A] ──► [B] (dead)
              [C]

- A continues; still has majority (A + C = 2/3)
- A marks B's log as pending; retries AppendEntries indefinitely
- B recovers → A sends all missing entries → B catches up
- No election; no term change
```

**Scenario 2: Leader crashes mid-write**
```
Term 1: Leader A appends entry at index 5
  A sends AppendEntries to B and C
  B ACKs; C hasn't received yet
  A crashes before committing

Term 2: B or C times out → election
  Winner: B (has index 5) OR C (missing index 5)

Case: C wins election (it didn't get index 5)
  C becomes leader term 2; C's log has index 1–4
  C sends AppendEntries to B → B sees leader term=2, leader log ends at 4
  B must TRUNCATE its index 5 (it was never committed → safe to discard)
```

**Key rule**: Raft never commits log entries from a previous term alone. A new leader commits a "no-op" entry for the current term first — this implicitly commits all prior entries.

### Raft Split-Brain Prevention via Terms

```
Scenario: network partition splits 5-node cluster into [A,B] and [C,D,E]

Term 3: A is leader
Partition occurs:
  Left side [A,B]: A tries to commit → only 2 nodes → no majority → A STALLS (cannot commit)
  Right side [C,D,E]: C,D,E time out → election → term 4 → D becomes new leader (3/5 majority)

A is now a stale leader (term 3). D is the real leader (term 4).

A tries to replicate to C → C replies with term=4 → A sees higher term → A steps down immediately
Split brain impossible: A cannot commit without majority, and any node on the majority side
carries term 4 which invalidates A's term 3 writes.
```

**Fencing property**: `term` acts as a fencing token. Any message with `term < current_term` is rejected. A stale leader's writes are never committed because:
1. It can't get majority ACKs (the majority is in the other partition)
2. If it somehow contacts the other partition, they reject it with their higher term

### Used In
etcd, CockroachDB, TiKV, Consul

---

## ZAB (Zookeeper Atomic Broadcast)

Used in Apache ZooKeeper. Technically atomic broadcast (not pure consensus) but maintains strong consistency. Open-source equivalent to Google Chubby.

Used indirectly by: HBase, Storm, Kafka (for controller election), HDFS NameNode HA.

### ZAB Epoch — ZAB's Equivalent of Raft Term

ZAB uses an **epoch** number (called `zxid` high 32 bits = epoch, low 32 bits = counter within epoch).

```
zxid = | epoch (32 bits) | counter (32 bits) |
         ^^^ changes on   ^^^ resets to 0
             leader change    each new epoch
```

Every write proposal carries the full `zxid`. A follower rejects any proposal with an older epoch.

### ZAB Node Failure Scenarios

**Scenario 1: Follower crashes**
```
Epoch 3: Leader L broadcasts proposal P to [F1] [F2] (F2 dead)
  F1 ACKs → majority (L + F1 = 2/3) → L commits P
  F2 recovers → connects to L → L sends diff (missing proposals) → F2 catches up
  zxid ensures F2 knows exactly which proposals it missed
```

**Scenario 2: Leader crashes (DISCOVERY phase)**
```
Epoch 3: Leader L crashes after sending proposal P to F1 but NOT F2

Phase 1 — DISCOVERY (new leader election):
  F1 and F2 both time out. Start election.
  Candidate with highest zxid wins (F1 has zxid epoch=3,counter=5; F2 has epoch=3,counter=4)
  F1 wins → new epoch = 4

Phase 2 — SYNCHRONIZATION:
  New leader F1 (epoch 4) tells F2: "you're missing proposal counter=5 from epoch 3"
  F1 sends P to F2 → F2 applies it
  Both now in sync at epoch 4, counter=0

Phase 3 — BROADCAST (normal operation):
  F1 is now leader for epoch 4; processes new writes
```

**Key difference from Raft**: ZAB has an explicit SYNCHRONIZATION phase where new leader ensures all followers have the same log before accepting new writes. Raft's new leader begins immediately and syncs lazily as it replicates.

### ZAB Split-Brain Prevention via Epoch

```
Partition: 3 ZooKeeper nodes [L] | [F1, F2]

Epoch 3: L is leader
  L sends proposal P (zxid = 3-5)
  F1 and F2 don't ACK → L cannot commit (no majority)
  F1 and F2 time out → election → epoch 4 → F1 becomes leader

L (still thinks it's epoch-3 leader) tries to send to F1:
  F1 replies: "my epoch is 4, yours is 3 → rejected"
  L sees higher epoch → steps down → reconnects as follower

Any client connected to old L during partition:
  L could not have committed (no majority acks) → no inconsistency
  L's uncommitted proposals are discarded on rejoin
```

**Zombie leader problem**: an old leader that gets isolated may still respond to reads (ZooKeeper clients). This is why ZooKeeper has a `sync()` call — forces a read to go through the current leader before returning, preventing stale reads from a zombie.

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
