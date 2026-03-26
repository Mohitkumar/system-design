# DDIA Chapter 5: Replication

## Theory

### Leader-Based Replication
- Writes only through leader → replicated to followers
- Reads from any replica (including leader)
- **Sync**: follower guaranteed up-to-date, but blocks if follower unresponsive
- **Async**: widely used; risk of data loss on leader failure
- **Semi-sync**: one follower sync, rest async — practical balance

### Adding a Follower
1. Snapshot leader DB
2. Copy snapshot to new follower
3. Follower requests all changes after snapshot point

### Failover Risks
- New leader may miss recent writes → data loss
- **Split brain**: old leader wakes up thinking it's still leader → two leaders accepting writes
- Wrong timeout → unnecessary failovers

### Replication Log Formats
| Type | How | Pros | Cons |
|------|-----|------|------|
| Statement-based | SQL sent as-is | Simple | Non-determinism (NOW(), RAND()) |
| Write-ahead log | Append-only byte log | Accurate | Coupled to storage engine version |
| Logical log | Row-level change records | Decoupled from engine | More bytes |
| Trigger-based | App code decides | Flexible | Overhead, bug-prone |

### Replication Lag Problems
| Problem | Definition | Fix |
|---------|-----------|-----|
| Read-after-write | User can't see own writes | Route own-data reads to leader |
| Monotonic reads | User sees data go backwards | Same replica per user |
| Consistent prefix | Causally related writes seen out of order | Same partition for causally related writes |

---

## Multi-Leader Replication
- One leader per datacenter
- **Pros**: hidden inter-DC latency, independent per-DC, tolerates network issues
- **Cons**: write conflicts, problematic with auto-increment keys/triggers/constraints
- **Avoid if possible**

### Conflict Resolution
- Last write wins (LWW) — data loss risk
- Higher replica ID wins
- Merge values
- Record conflict explicitly, resolve in app

### Topologies
- **All-to-all**: resilient, causality issues
- **Star/Circular**: single point of failure

---

## Leaderless Replication (Dynamo-style)
- Client writes/reads to **multiple replicas in parallel**
- On read: detect stale value → write newest back (read repair)
- Background anti-entropy process syncs differences

### Quorum
```
w + r > n
```
- `n` = total replicas, `w` = write acks needed, `r` = read replicas queried
- Smaller `w`/`r` → lower latency, higher availability, more stale reads

### Conflict Resolution
| Strategy | Trade-off |
|----------|-----------|
| Last write wins | Convergence, but data loss |
| Happens-before detection | Accurate, complex |
| Merge values | No loss, app decides display |
| Version vectors | Per-replica version numbers, detect concurrency |

---

## System Design Implications
- Use **leader-based** for mostly-read workloads
- Use **multi-leader** for multi-datacenter (cautiously)
- Use **leaderless** for write-intensive, high-availability needs
- Always plan for replication lag — don't assume strong consistency unless guaranteed
