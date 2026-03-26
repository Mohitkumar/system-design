# Consistency Models

## Consistency Spectrum (weakest → strongest)

```
Eventual → Causal → Monotonic Read/Write → Consistent Prefix → Snapshot → Linearizable
```

Most replicated DBs provide only **eventual consistency** by default.
Stronger guarantees = worse performance, easier correctness reasoning.

---

## Strong Consistency Models

### Linearizability
> System appears to have **one copy of data**, all operations are atomic with real-time ordering.

- Once any read returns a new value, **all subsequent reads** (any client) must also return that value
- Indistinguishable from a non-replicated single-node system
- Testable: record timings of all requests/responses, verify sequential ordering
- **Always slow** — even without network faults (coordination cost)

**Implementation support:**
| Replication Type | Linearizable? |
|-----------------|--------------|
| Single-leader (reads from leader) | Yes |
| Consensus algorithms (Raft, Paxos) | Yes |
| Multi-leader | No |
| Leaderless (Dynamo-style) | No |

**When required:**
- Leader election (one node must hold lock)
- Uniqueness constraints (usernames, IDs)
- Cross-channel timing dependencies (file uploaded → job triggered)

### Sequential Consistency
- Operations appear atomic and in an order consistent across all nodes
- Weaker than linearizable — no real-time ordering guarantee

---

## Weak Consistency Models

### Causal Consistency
- Causally related operations maintain order
- Concurrent operations can be in any order
- **Strongest model that doesn't slow down due to network failures**
- Linearizability implies causal consistency (strict superset)

### Client-Centric Models
| Model | Guarantee |
|-------|-----------|
| Read-your-writes | Client always sees own writes |
| Monotonic reads | Client never sees data go backwards |
| Monotonic writes | Client's writes applied in order |
| Consistent prefix | Client sees causally related writes in order |

### Eventual Consistency
- Replicas eventually agree on the same value if writes stop
- No timing guarantee
- Anomalies possible during normal operation

---

## Linearizability vs. Serializability

| Property | Type | Guarantee |
|----------|------|-----------|
| Serializability | Isolation (transactions) | Txs behave as if executed serially (no time constraint) |
| Linearizability | Recency (individual reads/writes) | Operations respect real-time ordering |

- 2PL and actual serial execution → **linearizable**
- SSI → **not linearizable** by design (optimistic, snapshot-based)
- Strict serializability = both linearizable + serializable

---

## ACID Isolation Levels

### Isolation Levels (weakest → strongest)

| Level | Prevents | Allows |
|-------|----------|--------|
| Read Uncommitted | Nothing | Dirty reads, phantom reads |
| Read Committed | Dirty reads/writes | Read skew, lost updates, phantom reads |
| Repeatable Read / Snapshot | + Read skew, lost updates | Phantom reads (some DBs) |
| Serializable | All anomalies | — |

### Concurrency Anomalies

| Anomaly | Description | Fix |
|---------|-------------|-----|
| Dirty read | Read another tx's uncommitted data | Read committed |
| Dirty write | Overwrite another tx's uncommitted data | Row-level locks |
| Read skew | See different snapshots mid-tx | Snapshot isolation (MVCC) |
| Lost update | Two concurrent read-modify-write, one lost | Atomic ops, explicit lock, auto-detect, CAS |
| Write skew | Two txs read same data, write different objects → invalid state | Serializable, `FOR UPDATE` |
| Phantom read | New rows appear mid-tx matching a query | Serializable, materialized conflicts |

---

## Snapshot Isolation (MVCC)

- Each tx reads from a **frozen snapshot** at start time
- Readers never block writers; writers never block readers
- DB keeps multiple object versions (MVCC)
- Prevents read skew — good for backups, analytics, integrity checks

```sql
-- Read committed: per-statement snapshot
-- Snapshot isolation: per-transaction snapshot
BEGIN;
  SELECT balance FROM accounts WHERE id = 1;  -- sees snapshot at BEGIN
  -- another tx writes balance=200 here
  SELECT balance FROM accounts WHERE id = 1;  -- still sees original value
COMMIT;
```

---

## Lost Update Solutions

```sql
-- Atomic write (best when expressible)
UPDATE counters SET value = value + 1 WHERE key = 'x';

-- Explicit lock
SELECT * FROM docs WHERE id = 1 FOR UPDATE;

-- Compare and set
UPDATE docs SET content = 'new' WHERE content = 'old';
```

| Solution | Notes |
|----------|-------|
| Atomic write | Best — database handles internally |
| Explicit lock (`FOR UPDATE`) | Works but easy to forget |
| Auto-detection | DB detects + aborts; works with snapshot isolation |
| CAS | May fail on old snapshot reads |
| Conflict resolution (replicated) | Allow conflicts, merge or LWW |

---

## CAP and Consistency Choice

- Systems **not** requiring linearizability → more tolerant of network partitions
- Causal consistency = strongest achievable without sacrificing availability under partitions
- **80/20 rule**: most apps only need read-your-writes + monotonic reads — much cheaper than full linearizability
