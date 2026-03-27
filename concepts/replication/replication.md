# Replication

## Why Replicate?
- Fault tolerance (survive node failures)
- Scalability (spread read load)
- Reduce latency (serve from nearby replica)

---

## Replication Algorithm Cost (by message count)
| Messages | Algorithm |
|----------|-----------|
| 1n | Async primary/backup |
| 2n | Sync primary/backup |
| 4n | 2-phase commit, Multi-Paxos |
| 6n | 3-phase commit, Paxos with leader election |

---

## Leader-Based (Primary/Backup)

- All writes go through leader → forwarded to followers
- Reads from any replica (including leader)

### Sync vs. Async
| Mode | Guarantee | Risk |
|------|-----------|------|
| Synchronous | Follower has up-to-date copy | Blocks if any follower unresponsive |
| Asynchronous | Low latency | Data loss on leader failure |
| Semi-sync | One follower sync, rest async | Practical balance |

### Replication Log Formats
| Type | How | Pros | Cons |
|------|-----|------|------|
| Statement-based | SQL sent as-is | Simple | Non-determinism (NOW(), RAND()) |
| Write-ahead log | Append-only byte log | Accurate | Coupled to storage engine version |
| Logical log | Row-level change records | Decoupled | More bytes |
| Trigger-based | App code decides | Flexible | Overhead, bug-prone |

### Adding a Follower
1. Snapshot leader DB
2. Copy snapshot to new follower
3. Follower requests all changes since snapshot (via log sequence number)
4. Follower catches up → serves requests

### Leader Failover Process

```
1. Detect failure     — heartbeat timeout (typically 30s); avoid false positives from GC pauses
2. Elect new leader   — replica with most up-to-date replication offset wins
                        (Raft: candidate requests votes; needs majority)
                        (Sentinel: ≥ quorum Sentinels agree)
3. Reconfigure clients— old leader connections rejected; clients redirect to new leader
4. Old leader rejoins — must be demoted to follower; must discard any writes not replicated
```

| Method | How | Used By |
|--------|-----|---------|
| **Raft election** | Followers time out → become candidate → request votes → majority = leader | etcd, CockroachDB, TiKV |
| **Sentinel** | External watchers monitor leader; quorum vote to promote replica | Redis Sentinel |
| **Manual failover** | Operator promotes replica | PostgreSQL (default), MySQL |

### Failover Risks & Mitigations

| Risk | What happens | Mitigation |
|------|-------------|------------|
| **Data loss** | New leader lacks writes not yet replicated from old leader | Semi-sync replication — wait for ≥1 follower ACK before returning success |
| **Split brain** | Old leader revives, both accept writes | Fencing: epoch/generation number; old leader rejects writes if it sees higher epoch; STONITH (shoot the other node) |
| **False positive** | High load / GC pause looks like failure → unnecessary failover | Tunable timeout + hysteresis (require N consecutive misses, not just 1) |
| **Replica too far behind** | Promoted replica diverges significantly → data loss even with election | Set max replication lag threshold; don't elect replicas lagging >N bytes/ms |

---

## Replication Lag Problems

| Problem | Definition | Fix |
|---------|-----------|-----|
| Read-after-write | User can't see own submitted writes | Route own-data reads to leader; track logical timestamp |
| Monotonic reads | User sees data go "backwards in time" | Same replica per user session |
| Consistent prefix | Causally related writes seen out of order | Same partition for causally related writes |

**Rule**: If you can't tolerate minutes of lag, use stronger consistency guarantees.

---

## Multi-Leader Replication

One leader per datacenter.

**Pros**: hidden inter-DC latency, independent per-DC operation, tolerates network issues
**Cons**: write conflicts, breaks auto-increment keys/triggers/constraints

**Avoid if possible** — dangerous and complex.

### Conflict Resolution
| Strategy | Trade-off |
|----------|-----------|
| Last write wins (LWW) | Simple, data loss risk |
| Higher replica ID wins | Deterministic, data loss |
| Merge values | No loss, app complexity |
| Explicit conflict record | Lossless, requires app handling |

### Topologies
| Topology | Risk |
|----------|------|
| All-to-all | Causality issues (some links faster) |
| Star / Circular | Single point of failure |

---

## Leaderless Replication (Dynamo-style)

Client sends writes/reads to **multiple replicas in parallel**. No single leader.

### Quorum Formula
```
w + r > n
```
`n` = replicas, `w` = write acks required, `r` = replicas read

| Config (n=3) | Write | Read | Trade-off |
|-------------|-------|------|-----------|
| w=3, r=1 | Strong durability | Fast read | Slow writes |
| w=2, r=2 | Balanced | Balanced | Default |
| w=1, r=3 | Fast write | Strong read | Risky writes |

**Sloppy quorum**: Dynamo allows writes on replacement nodes during failures → breaks quorum overlap → **not truly consistent** — only probabilistic.

### Conflict Detection on Read
- Read R nodes → compare versions
- Discard strictly older (via vector clock)
- If concurrent versions (incomparable clocks) → return all → app resolves

### Read Repair + Anti-Entropy
- **Read repair**: on read, detect stale replica → write newest value back
- **Anti-entropy**: background process constantly compares and syncs differences

### Leaderless "Failover" — No Election Needed

Leaderless systems have no leader to fail over from. Instead, availability is structural:

```
Normal (n=3, w=2, r=2):
  Client writes to all 3 → 2 ACK → success
  Node A goes down → write to B + C → still meets w=2

Node A recovers:
  Stale → read repair catches up A on next read
  Or: anti-entropy background sync catches up A
```

| Scenario | What happens | Recovery |
|----------|-------------|----------|
| **1 of 3 nodes down** | Writes/reads still meet quorum (w=2, r=2) | Node rejoins → read repair + anti-entropy syncs missed writes |
| **2 of 3 nodes down** | Quorum unmet → writes/reads fail (CP behavior) | Wait for nodes to recover; or relax to sloppy quorum |
| **Network partition** | Two sides each below quorum → reject | Each side rejects; no split-brain writes (unlike multi-leader) |
| **Sloppy quorum** | Accept writes on available nodes even if not the "home" N | Hinted handoff: writes stored temporarily on substitute; forwarded when target recovers |

**Hinted handoff**: substitute node stores write with a hint `"this belongs to node A"`; when A recovers, substitute forwards the write. Improves availability but breaks quorum overlap guarantees — only **best-effort** durability.

**Anti-entropy with Merkle trees**: each node builds a Merkle tree of its key range; nodes exchange tree hashes to find differing subtrees efficiently → sync only divergent keys (O(diff) not O(total)).

---

## Weak Consistency / Eventual Consistency (Dynamo-style Systems)

Strong consistency is expensive — information travels at light speed, majority coordination per op is costly.

### Two Categories of Eventual Consistency
| Category | Guarantee | Example |
|----------|-----------|---------|
| Probabilistic | May see anomalies during normal ops; detects conflicts later | Amazon Dynamo |
| Strong eventual | Convergence to equivalent sequential execution; no anomalies | CRDTs |

### Probabilistically Bounded Staleness (PBS)
Research by Bailis et al. (2012): eventually consistent stores are often consistent within **tens to hundreds of milliseconds** in practice.

Example (Cassandra): R=1/W=1 → inconsistency window ~1352ms; R=2/W=1 → ~202ms.

---

## CRDTs (Convergent Replicated Data Types)

Operations that are **commutative + associative + idempotent** converge regardless of order — no coordination needed.

### Three Required Properties
- **Associative**: `(a+(b+c)) = (a+b)+c`
- **Commutative**: `a+b = b+a`
- **Idempotent**: `a+a = a`

These form a **join semilattice** — any data type expressible as a semilattice can guarantee convergence.

### CRDT Types
| Type | Merge Rule | Notes |
|------|-----------|-------|
| G-Counter | Sum per-node counters | Grow-only |
| PN-Counter | Increments − decrements | Two G-counters |
| G-Set | Union | No removal |
| 2P-Set | Add-set ∪ Remove-set | Removes win |
| OR-Set | Track add/remove pairs | Removes only win if they saw the add |
| LWW-Register | max(timestamp) wins | Data loss possible |
| MV-Register | Keep concurrent versions | App merges |

### Real Systems
| System | Usage |
|--------|-------|
| Redis Enterprise (Active-Active) | CRDT counters for geo-distributed deployments |
| Riak | OR-Set for shopping cart, PN-Counter |
| Apple iCloud Notes | Text CRDT for offline merge |
| Figma | CRDT-like OT for concurrent shape edits |

---

## CALM Theorem

> "Logically monotonic programs are guaranteed to be eventually consistent."

- **Monotonic**: new information never invalidates prior conclusions — safe without coordination
- **Non-monotonic**: new knowledge can invalidate conclusions — requires coordination

### Monotonic (no coordination needed)
Selection, projection, join, union, recursive Datalog

### Non-monotonic (coordination required)
Negation, set difference, aggregation, universal quantification

**Key insight**: coordination protocols ARE aggregations — 2PC requires unanimous votes, Paxos requires majority votes. Aggregation = non-monotonic = requires coordination.

**Practical**: many computations can run coordination-free and only need coordination when passing results to external systems.

---

## Replica Synchronization Methods

| Method | How | Trade-off |
|--------|-----|-----------|
| Gossip | Random peer selection every t seconds | Scalable, no SPOF, probabilistic only |
| Merkle trees | Hash content at multiple granularities; exchange only diff | Efficient key comparison across nodes |

---

## Summary: Replication Model Selection

| Model | Best For | Watch Out |
|-------|----------|-----------|
| Single-leader (async) | Read-heavy, tolerate lag | Data loss on leader failure |
| Single-leader (sync) | Strong durability | Blocks on slow followers |
| Multi-leader | Multi-datacenter writes | Write conflicts, complexity |
| Leaderless | High availability, write-intensive | Eventual consistency, conflicts |
| CRDTs | Auto-merging without coordination | Limited to specific data shapes |
