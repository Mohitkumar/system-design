# Transactions

## ACID — What Each Property Actually Means

| Property | Meaning | Who enforces it |
|----------|---------|-----------------|
| **Atomicity** | All writes in a tx commit or none do. On fault → abort → client retries safely. NOT about concurrency — about all-or-nothing on crash. | Database (WAL + undo log) |
| **Consistency** | DB moves from one valid state to another (invariants hold: no negative balance, FK constraints intact). | **Application** defines the invariants; DB provides the tools |
| **Isolation** | Concurrently executing txs appear to run serially — no interference. Most DBs provide weaker guarantees by default. | Database (locks, MVCC) |
| **Durability** | Committed data survives crashes — flushed to disk (WAL fsync) or replicated to N nodes. | Database (storage + replication) |

---

## Single-Object vs Multi-Object Operations

### Single-Object
- One row/document; always atomic in any serious DB
- `INCR` in Redis, `findAndModify` in MongoDB — atomic by design
- Storage engines use WAL + crash recovery to guarantee atomicity per object

### Multi-Object
- Required for: foreign keys, denormalized copies, secondary index updates
- Without a transaction: partial write leaves DB inconsistent (FK exists but referencing row absent)
- NoSQL often skips this → app must handle partial failure
- **Retry pitfall**: network failure after commit looks like failure → double-execute → need idempotency key

---

## Isolation Levels (weakest → strongest)

```
Read Uncommitted  ←  allows dirty reads
Read Committed    ←  no dirty reads/writes; default in PostgreSQL, Oracle, SQL Server
Repeatable Read   ←  no non-repeatable reads; snapshot per tx; default in MySQL InnoDB
Snapshot Isolation←  consistent snapshot for entire tx (MVCC); no write skew prevention
Serializable      ←  full isolation; as if txs ran one at a time
```

---

## Weak Isolation — Problems & Fixes

### Dirty Reads
**What**: reading uncommitted data from another in-flight tx.
```
Tx A: UPDATE accounts SET balance = 0 WHERE id=1   -- not yet committed
Tx B: SELECT balance FROM accounts WHERE id=1       -- sees 0 (wrong!)
Tx A: ROLLBACK
```
**Fix**: Read Committed isolation — only read committed values (DB maintains two versions: before-update value served to readers).

---

### Dirty Writes
**What**: overwriting another tx's uncommitted write.
```
Tx A: UPDATE listings SET buyer=Alice WHERE id=1    -- not committed
Tx B: UPDATE listings SET buyer=Bob   WHERE id=1    -- overwrites Alice (dirty write)
Tx A commits → Alice thinks she won; Bob's invoice also created
```
**Fix**: Row-level locks — writer acquires lock; other writers wait until commit/abort.

---

### Read-After-Write (Non-Repeatable Read)
**What**: reading the same row twice in one tx returns different values (another tx committed in between).
```
Tx A: SELECT balance → 100
Tx B: UPDATE balance = 50; COMMIT
Tx A: SELECT balance → 50   -- different value, same tx!
```
**Fix**: Repeatable Read — tx reads from its own snapshot (MVCC). Same query always returns same result within the tx.

---

### Snapshot Isolation (MVCC Implementation)
Each tx gets a consistent snapshot of the DB at start time. Reads never block writers; writers never block readers.

**MVCC mechanics:**
```
Each row has: created_by (tx_id), deleted_by (tx_id or null)

Tx 13 starts → snapshot = {committed txs with id < 13} + {txs already committed before 13 started}

On read: return row if:
  - created_by tx is committed AND created_by < current_tx_id
  - deleted_by is NULL OR deleted_by tx is not yet committed
```

**Visibility rules:**
- Row created by a later tx → invisible (not in my snapshot)
- Row deleted by a later tx → still visible (delete not in my snapshot)
- Row deleted by an earlier committed tx → invisible

**Used by**: PostgreSQL, MySQL InnoDB, Oracle, CockroachDB, TiKV.

---

### Phantom Reads
**What**: a query returns different rows on two executions within the same tx (another tx inserted/deleted matching rows).
```
Tx A: SELECT * FROM bookings WHERE room=101 AND checkin='2024-01-01' → 0 rows
Tx B: INSERT INTO bookings (room, checkin) VALUES (101, '2024-01-01'); COMMIT
Tx A: SELECT same query → 1 row   -- phantom appeared!
Tx A: INSERT booking thinking room is free → double booking
```
**Fix options**:
- **Predicate lock**: lock all rows matching the WHERE clause (including not-yet-existing rows) — expensive
- **Index-range lock**: lock the index range covering the query → cheaper; blocks insertions into that range
- **Serializable Snapshot Isolation**: detects the conflict at commit time without locking

---

## Write Skew
**What**: two txs read the same data, make decisions based on it, then write to different objects — breaking an invariant. Neither tx wrote dirty data, but the combination is invalid.

```
Invariant: at least 1 doctor must be on-call

Tx Alice: SELECT count(*) FROM oncall → 2  (both Alice and Bob)
Tx Bob:   SELECT count(*) → 2
Tx Alice: UPDATE oncall SET oncall=false WHERE name='Alice'  -- sees 2, thinks it's safe
Tx Bob:   UPDATE oncall SET oncall=false WHERE name='Bob'    -- sees 2, thinks it's safe
Both commit → 0 doctors on call ← invariant violated
```

**Pattern**: read → decide → write (where write is to a different object than what was read).

**Fix**: make writes conflict:
1. Materialize the conflict — create a lock row that both txs must write to
2. `SELECT FOR UPDATE` — explicitly lock the read rows
3. Serializable isolation (SSI or 2PL with predicate locks)

**Real examples**: meeting room double-booking, username uniqueness, balance ≥ 0 across accounts, claiming a unique item.

---

## Preventing Concurrency Bugs

### Compare-and-Set (CAS)
Atomic conditional write — only updates if current value matches expected.
```sql
UPDATE wiki SET content = 'new' WHERE id=1 AND content = 'old';
-- Returns 0 rows affected → someone else changed it → retry
```
- Works only on single objects
- Unsafe if DB reads from old snapshot (CAS check passes on stale read)

### Explicit Locking (`SELECT FOR UPDATE`)
```sql
BEGIN;
SELECT * FROM bookings WHERE room=101 FOR UPDATE;  -- acquires row lock
-- safe to check and insert; other txs block on this row
INSERT INTO bookings ...;
COMMIT;
```
- Prevents write skew on existing rows
- Does NOT prevent phantom inserts (lock only covers rows that exist)

### Materializing Conflicts
When there are no rows to lock (phantom problem), pre-create lock rows:
```sql
-- Pre-populate all room×timeslot combinations
INSERT INTO room_slots (room, date) VALUES (101, '2024-01-01'), ...;

-- Tx acquires lock on the slot row:
SELECT * FROM room_slots WHERE room=101 AND date='2024-01-01' FOR UPDATE;
```
Last resort — leaks concurrency control concern into data model.

---

## Serializability Implementations

### Actual Serial Execution
- Single thread processes one tx at a time — eliminates all concurrency bugs
- Feasible only with fast in-memory txs + stored procedures (no network round trips mid-tx)
- Scale: partition data → each partition has own thread → cross-partition txs = expensive coordination
- **Used by**: VoltDB, Redis (single-threaded), H-Store, Datomic

### Two-Phase Locking (2PL)
Pessimistic approach — block before conflict can occur.

```
Phase 1 (growing): acquire locks; never release
Phase 2 (shrinking): release locks; never acquire new

Read  → shared lock    (multiple readers OK; blocks writers)
Write → exclusive lock (blocks all readers AND writers)
```

**Deadlock**: Tx A holds lock on row 1, wants row 2. Tx B holds row 2, wants row 1.
- DB detects cycle in wait-for graph → aborts one tx → other proceeds
- Application must retry aborted tx

**Predicate locks**: lock all rows matching a WHERE clause — prevents phantoms but expensive to check.

**Index-range locks**: lock a range in the index covering the predicate (broader but cheaper):
```
Query: WHERE room=101 AND checkin BETWEEN '2024-01' AND '2024-02'
→ Lock index entry for room=101 (or the full time range in that index)
→ Any INSERT matching room=101 must acquire compatible lock → blocks
```

**Performance**: significantly worse than snapshot isolation; writers block readers; unstable latencies.

### SSI (Serializable Snapshot Isolation)
Optimistic approach — run without blocking; detect conflicts at commit.

**Premise tracking:**
```
Tx reads rows matching a condition → DB records the read premise: "I read X at time T"

At commit:
  Was X modified by another committed tx after T?
  YES → my read is stale → abort + retry
  NO  → commit succeeds
```

**Two types of conflicts detected:**
1. **Stale MVCC read**: tx read a value, but another tx committed a write to that value before this tx started (tx ignored that write because it wasn't in its snapshot)
2. **Reads affected by a write**: another tx commits a write that changes the result of a previously executed query

**Used by**: PostgreSQL (since 9.1), CockroachDB, FoundationDB, TiKV (Percolator-based)

---

## Pessimistic vs Optimistic Concurrency

| | Pessimistic (2PL) | Optimistic (SSI, OCC) |
|--|---|---|
| **Assumption** | Conflicts are likely — block early | Conflicts are rare — detect late |
| **Blocking** | Yes — readers/writers wait for locks | No — all txs run concurrently |
| **Abort rate** | Low (conflicts prevented) | High under contention (abort + retry) |
| **Throughput** | Lower (lock overhead + blocking) | Higher under low contention |
| **Latency** | Higher (wait for locks) | Lower (no waiting) |
| **Best for** | High contention, short critical sections | Low contention, read-heavy workloads |
| **Starvation** | Possible (long-held locks) | Possible (repeated aborts under contention) |

---

## Summary: Which Isolation Level Prevents What

| Problem | Read Committed | Repeatable Read / Snapshot | Serializable |
|---------|:--------------:|:--------------------------:|:------------:|
| Dirty reads | ✅ | ✅ | ✅ |
| Dirty writes | ✅ | ✅ | ✅ |
| Non-repeatable reads | ❌ | ✅ | ✅ |
| Phantom reads | ❌ | ❌ (MVCC only) | ✅ |
| Write skew | ❌ | ❌ | ✅ |
| Lost updates | ❌ | Some DBs ✅ | ✅ |

---

## Transaction Isolation Decision Tree

```
Need serializability?
  ├─ Low contention, distributed → SSI (PostgreSQL, CockroachDB)
  ├─ Simple single-node OLTP    → Actual serial execution (VoltDB)
  └─ High contention, locks OK  → 2PL with index-range locks

Need snapshot reads without serializability?
  └─ Snapshot Isolation / Repeatable Read (MVCC)

Need just no dirty reads?
  └─ Read Committed (cheapest, PostgreSQL default)

Application can handle conflicts?
  └─ Optimistic (CAS, version field) + retry loop
```

---

## Distributed Transactions

Multi-object txs across partitions/nodes — requires coordination.

**Why NoSQL skipped them**: 2PC latency + availability cost (coordinator is SPOF).
**When unavoidable**: cross-shard FK, cross-service atomic write.

→ See [consensus/two_phase_commit.md](../consensus/two_phase_commit.md) for 2PC details.
→ See [consistency/percolator_protocol.md](percolator_protocol.md) for 2PC without coordinator (TiKV).
