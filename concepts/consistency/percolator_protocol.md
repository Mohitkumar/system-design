# Percolator Protocol (Distributed MVCC Transactions)

## Background

Google Percolator (2010) — incremental processing on top of Bigtable. First practical implementation of **lock-free distributed MVCC transactions** without a dedicated transaction coordinator.

Used by: **TiKV**, Google Spanner (inspired), CockroachDB (similar approach).

---

## Key Insight

Traditional 2PC requires a coordinator that holds locks. Percolator eliminates the coordinator by:
1. Using a **primary lock** as the single source of truth for commit/abort
2. Letting any node resolve a transaction by checking the primary lock's state
3. Using **MVCC timestamps** (from TSO) to order operations without locks on reads

---

## Storage Layout (TiKV Column Families)

| CF | Key | Value | Role |
|----|-----|-------|------|
| `CF_DEFAULT` | `(key, start_ts)` | value bytes | Actual data versions |
| `CF_LOCK` | `key` | lock info + primary key ref | In-flight tx locks |
| `CF_WRITE` | `(key, commit_ts)` | `{type, start_ts}` | Committed version index |

Timestamps stored in **descending order** → RocksDB forward scan = newest version first.

---

## Transaction Protocol

### Phase 1 — Prewrite

```
1. Get start_ts from TSO (Timestamp Oracle)

2. Choose one key as "primary" lock, rest are "secondary"

3. For each key to write:
   a. Check CF_WRITE at (key, ts > start_ts)
      → if exists: write conflict → abort
   b. Check CF_LOCK at key
      → if exists (any ts): lock conflict → abort or wait
   c. Write value:  CF_DEFAULT[(key, start_ts)] = value
   d. Write lock:   CF_LOCK[key] = {start_ts, primary_key, ttl, ...}
```

### Phase 2 — Commit

```
1. Get commit_ts from TSO

2. Check primary lock still exists at CF_LOCK[primary_key]
   → if missing: transaction was rolled back by someone else → abort

3. Write to CF_WRITE[primary_key, commit_ts] = {Put, start_ts}
   Delete CF_LOCK[primary_key]
   ← THIS IS THE COMMIT POINT — atomic, single write

4. Async: clean up secondary locks (commit secondaries)
   For each secondary key:
     Write CF_WRITE[(key, commit_ts)] = {Put, start_ts}
     Delete CF_LOCK[key]
```

**Correctness**: commit is a single atomic write to CF_WRITE for the primary key. Secondary cleanup can happen lazily — readers will resolve it.

---

## Read Protocol

```
Read key K at snapshot timestamp T:

1. Check CF_LOCK[K]
   → if lock.start_ts < T: another tx has an in-flight write visible to us
     → may need to wait or resolve (see below)
   → if lock.start_ts > T: ignore (started after our snapshot)

2. Scan CF_WRITE for (K, commit_ts ≤ T) newest first
   → get start_ts from write record

3. Fetch CF_DEFAULT[(K, start_ts)]
   → return value
```

---

## Lock Conflict Resolution

When a reader encounters a lock at `start_ts < T`:

```
Check primary lock status:
  Case 1: Primary committed (CF_WRITE has entry)
    → Secondary is dangling → commit it (write CF_WRITE, delete CF_LOCK)
    → Read the committed value

  Case 2: Primary not committed + lock TTL expired
    → Transaction is dead → rollback it
    → Write CF_WRITE[(primary, start_ts)] = {Rollback}
    → Delete CF_LOCK[primary]
    → Clean up secondaries

  Case 3: Primary lock still live (TTL not expired)
    → Transaction is still in progress → wait or retry
```

Any node can resolve any transaction — **no dedicated coordinator needed**.

---

## Timestamp Oracle (TSO)

Central monotonically increasing timestamp generator.

```
64-bit timestamp = [46-bit physical ms | 18-bit logical counter]
```

- Pre-allocates ranges to disk; serves from memory (no disk I/O per request)
- Clients batch requests (one RPC per batch) to reduce overhead
- Single node — leader in a Raft group (PD in TiKV) for HA
- Failure: new leader resumes from a timestamp > last persisted batch → no duplicate timestamps

**Bottleneck**: all transactions need 2 TSO calls (start_ts + commit_ts). Mitigated by batching.

---

## MVCC Garbage Collection

```
GC safe point = min(start_ts of all active transactions)

All versions with commit_ts < GC safe point can be deleted
→ Triggered during RocksDB compaction
→ PD tracks and broadcasts GC safe point to all TiKV nodes
```

---

## Percolator vs. Traditional 2PC

| | Traditional 2PC | Percolator |
|-|----------------|-----------|
| Coordinator | Required, holds locks | None — primary lock is coordinator |
| Blocking on coordinator failure | Yes | No — any node resolves via primary lock |
| Lock visibility | Coordinator memory | CF_LOCK in storage (persistent) |
| Commit point | Coordinator decision log | Single CF_WRITE entry for primary |
| Read blocking | Readers wait for locks | Readers resolve stale locks lazily |
| Clock dependency | No | TSO required for ordering |

---

## Percolator in CockroachDB (Similar Approach)

CockroachDB uses **write intents** (same concept as locks in CF_LOCK):
- Intents = uncommitted MVCC values flagged with transaction ID
- Transaction record = equivalent of primary lock
- Commit = single write to transaction record → `COMMITTED`
- Intent cleanup = async, correctness doesn't depend on it
- Uses HLC instead of TSO (no single-node timestamp bottleneck)
