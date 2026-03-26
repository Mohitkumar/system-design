# Two-Phase Commit (2PC) and Distributed Transactions

## Two-Phase Commit (2PC)

Atomic commit across multiple nodes/partitions.

### Protocol
```
Phase 1 — Prepare:
  Coordinator → all participants: "Are you ready to commit?"
  Each participant: durably stores data, votes "Yes" or "No"

Phase 2 — Decision:
  All voted Yes → Coordinator → "Commit" (irrevocable)
  Any voted No  → Coordinator → "Abort"
```

**Two irrevocable points:**
1. Participant votes Yes → committed to commit if asked
2. Coordinator sends decision → irreversible (can be undone by compensating tx, not rollback)

### 2PC Problems

| Problem | Description |
|---------|-------------|
| Coordinator crash after prepare | Participants hold locks, blocked until coordinator recovers |
| Participant crash after Yes vote | Must wait for coordinator — can't unilaterally abort |
| Network partition | May cause indefinite blocking |
| Performance | ~10× slower than single-node tx |

- Participants hold **locks while waiting** → blocks larger parts of the application
- Fix: manual human intervention or **heuristic decisions** (risky — may break atomicity)

### 2PC is CA (not partition-tolerant)
- If coordinator crashes mid-protocol → system blocks indefinitely
- 3PC theoretically solves this but impractical (assumes synchronous network)

---

## Distributed Transaction Types

| Type | Description | Optimization |
|------|-------------|-------------|
| Internal | Same DB software across all nodes | Can use optimized internal protocol |
| Heterogeneous (XA) | Different systems, standard XA protocol | Limited optimization; broad compatibility |

**XA** (eXtended Architecture): standard for heterogeneous distributed transactions. Supported by most relational DBs and message brokers. Uses 2PC internally.

---

## Heuristic Decisions

When coordinator is unreachable for too long:
- Participants can make a **heuristic decision** to commit or abort unilaterally
- Violates atomicity — some nodes commit, others abort
- Last resort — requires human review

---

## When to Use 2PC vs. Avoid It

**Use when:**
- Strong cross-shard atomicity required
- Heterogeneous systems (XA)
- Low-frequency, high-value operations

**Avoid when:**
- High-throughput hot path
- Can redesign with idempotency + eventual consistency
- Can scope writes to a single partition

**Alternative patterns:**
- Saga pattern (compensating transactions, no distributed lock)
- Outbox pattern (atomic local write + async event publishing)
- Single-partition design (avoid cross-shard writes)

---

## Coordinator Recovery

- Coordinator writes decision to **durable log** before sending commit/abort
- On recovery: re-read log, re-send decision to all participants
- Participants must be **idempotent** to handle duplicate commit/abort messages

---

## 2PC vs. Consensus (Paxos/Raft)

| | 2PC | Paxos/Raft |
|-|-----|-----------|
| Agreement type | Unanimous (all must agree) | Majority |
| Partition tolerant | No — blocks on any node failure | Yes |
| Leader failure | Blocks entire protocol | New leader elected |
| Use case | Distributed atomic commit | Leader election, replication |
| Latency | High (N-of-N round trips) | Lower (majority quorum) |
