# Transactions

## ACID Properties

| Property | Meaning | Owner |
|----------|---------|-------|
| Atomicity | All or nothing — on fault, entire tx aborts, client retries safely | Database |
| Consistency | DB in "good state" before/after | **Application** |
| Isolation | Concurrent txs don't see each other's partial writes | Database |
| Durability | Committed data survives crashes (disk write or N-node copy) | Database |

**Multi-object transactions needed for**: foreign keys, denormalized data, secondary indexes.

**Retry gotchas**: double execution on network failure (idempotency required); pointless on constraint violations.

---

## Serializability Implementations

| Approach | How | Throughput | Latency |
|----------|-----|-----------|---------|
| **Actual serial execution** | Single-threaded, one tx at a time | Low (single CPU) | Predictable |
| **Two-Phase Locking (2PL)** | Shared/exclusive locks; deadlock detection | Low | Unstable (deadlock aborts) |
| **SSI** (Serializable Snapshot Isolation) | Optimistic; conflict check at commit | High | Low (no blocking) |

### Actual Serial Execution
- Feasible with cheap RAM + short OLTP txs
- Use **stored procedures** — submit full tx code upfront, no network round trips
- Partition by key for multi-CPU scale (only if tx touches one partition)
- Used by: VoltDB, Redis (single-threaded), H-Store

### Two-Phase Locking (2PL)
- Read → shared lock; Write → exclusive lock
- Writers block readers AND writers (unlike MVCC)
- Deadlocks auto-detected → one tx aborted
- **Predicate locks**: lock all rows matching a query condition (prevents phantoms)
- **Index-range locks**: coarser but cheaper predicate locking
- Performance significantly worse; unstable latencies under contention

### SSI (Serializable Snapshot Isolation)
- Each tx reads from snapshot (MVCC); no blocking during execution
- At commit: check if any read premise is now stale (another tx committed a conflicting write)
- Abort if stale → retry
- Works well under **low contention**; degrades under high contention
- Distributable across partitions and nodes
- Used by: PostgreSQL (since 9.1), CockroachDB, FoundationDB

---

## Transaction Isolation Decision Tree

```
Need serializability?
  ├─ Low contention, distributed → SSI
  ├─ Simple single-node OLTP    → Actual serial execution
  └─ High contention, locks OK  → 2PL

Just need snapshot reads?
  └─ Snapshot Isolation (MVCC)

Just need no dirty reads?
  └─ Read Committed
```

---

## Distributed Transactions

Multi-object transactions across partitions or nodes — much harder.

**Why many NoSQL skipped them**: difficult to implement across partitions, performance cost.

**When unavoidable**: foreign keys spanning shards, atomic cross-service writes.

→ See [consensus/two_phase_commit.md](../consensus/two_phase_commit.md) for 2PC details.
