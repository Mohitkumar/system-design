# DDIA Chapter 7: Transactions

## Theory

### ACID
| Property | Meaning | Owner |
|----------|---------|-------|
| Atomicity | All or nothing — on fault, entire tx aborts | Database |
| Consistency | DB in "good state" before/after | **Application** |
| Isolation | Concurrent txs don't see each other's partial writes | Database |
| Durability | Committed data survives crashes | Database |

- Multi-object transactions needed for: foreign keys, denormalized data, secondary indexes
- Retrying aborted txs: not always safe — double execution on network failure, pointless on constraint violations

---

## Isolation Levels (weakest → strongest)

### Read Committed
- **Prevents**: dirty reads, dirty writes
- Dirty read: reading another tx's uncommitted data
- Dirty write: overwriting another tx's uncommitted write
- Implementation: row-level locks for writes; serve old value until commit for reads

### Snapshot Isolation (MVCC)
- Each tx reads from a **frozen snapshot** at start time
- Prevents: **read skew** (reading parts of DB at different points in time)
- Readers never block writers; writers never block readers
- DB keeps multiple object versions (MVCC)
- Use case: backups, analytic queries, integrity checks

### Serializable
- Strongest — prevents all race conditions

---

## Concurrency Problems Cheatsheet

| Problem | What happens | Fix |
|---------|-------------|-----|
| Dirty read | Read uncommitted data | Read committed isolation |
| Dirty write | Overwrite uncommitted data | Row-level locks |
| Read skew | See different snapshots mid-tx | Snapshot isolation (MVCC) |
| Lost update | Two concurrent read-modify-write, one lost | Atomic ops, explicit lock, auto-detect, CAS |
| Write skew | Two txs read same data, write different objects, invalid state | Serializable isolation, `FOR UPDATE` |
| Phantom read | New rows appear mid-tx | Serializable isolation, materialized conflicts |

---

## Lost Update Solutions
```sql
-- Atomic write (best if possible)
UPDATE counters SET value = value + 1 WHERE key = 'x';

-- Explicit lock
SELECT * FROM docs WHERE id = 1 FOR UPDATE;

-- Compare and set
UPDATE docs SET content = 'new' WHERE content = 'old';
```
- Auto-detection: DB detects lost update, aborts and retries (works with snapshot isolation)

---

## Serializability Implementations

| Approach | How | Throughput | Latency |
|----------|-----|-----------|---------|
| Actual serial execution | Single-threaded, one tx at a time | Low (single CPU) | Predictable |
| Two-Phase Locking (2PL) | Shared/exclusive locks; deadlock detection | Low | Unstable (deadlocks) |
| SSI (Serializable Snapshot Isolation) | Optimistic; conflict check at commit | High | Low (no blocking) |

### Actual Serial Execution Notes
- Feasible with cheap RAM + short OLTP txs
- Use **stored procedures** — submit full tx code upfront, no round trips
- Partition by key to scale across CPUs (only works if tx uses one partition)

### 2PL Notes
- Read → shared lock, Write → exclusive lock
- Writers block readers AND writers
- Performance significantly worse than weak isolation

### SSI Notes
- No locks during execution — detect conflicts at commit time
- Abort if stale premise detected (read value changed by another committed tx)
- Works well under low contention; degrades under high contention
- Distributable across partitions and nodes

---

## System Design Implications
- Default to **snapshot isolation** for most OLTP workloads
- Use **serializable** when correctness is critical and throughput allows
- Avoid multi-object txs in distributed NoSQL unless necessary — design around them
- SSI preferred over 2PL for modern distributed systems
