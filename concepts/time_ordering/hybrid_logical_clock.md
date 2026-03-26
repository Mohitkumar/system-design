# Hybrid Logical Clock (HLC)

## Problem with Physical Clocks

- Wall clocks can jump backward (NTP corrections)
- Different nodes have different views of "now" — clock skew
- Cannot use timestamps alone to establish happened-before ordering

## Problem with Logical Clocks (Lamport)

- Lose relationship to real time entirely
- Cannot answer "when did this happen?" in wall-clock terms
- Hard to implement TTL, lease expiry, etc.

---

## Hybrid Logical Clock (HLC)

Combines physical time + logical counter. Gives you:
- **Causality tracking** (like Lamport clocks)
- **Approximate wall-clock time** (like physical clocks)
- **Always moves forward** (no backward jumps)

### Structure
```
HLC = (physical_component, logical_component)
    = (≈ wall_clock_ms, tiebreaker_counter)
```

### Rules
1. On **local event**: `if wall_clock > physical: physical = wall_clock, logical = 0`
   else: `logical += 1`
2. On **send message**: attach current HLC
3. On **receive message**: `physical = max(local_physical, msg_physical, wall_clock)`
   then: `if physical unchanged: logical = max(local_logical, msg_logical) + 1`
   else: `logical = 0`

### Properties
- `HLC(a) < HLC(b)` → a happened before b (or concurrent)
- Physical component always ≥ wall clock (never goes back)
- Logical component bounded — doesn't grow unboundedly
- Clock skew bounded by max clock offset ε (typically 250ms in CockroachDB)

---

## HLC in CockroachDB

- Every node maintains an HLC
- Every read/write updates local HLC
- Every RPC carries sender's HLC → receiver advances its HLC on receipt
- **Guarantee**: all data on a node has HLC timestamp < next HLC value issued on that node

### Clock Uncertainty Window
```
Transaction at node A with timestamp T:
  → may have missed writes from node B with timestamps in (T, T+ε)
  → ε = max clock offset (CockroachDB default: 500ms)
```

**Uncertainty restarts**: if a read encounters a value with timestamp in `(T, T+ε)`:
1. Restart transaction with timestamp just above the conflicting value
2. Mark that node as "certain" → no more uncertainty restarts from it
3. Worst case: 1 restart per node touched (bounded)

### Causality Tokens
- App can pass a token from Txn A's commit to Txn B
- Ensures Txn B gets a timestamp > Txn A's commit timestamp
- Opt-in external consistency without full Spanner-style commit wait

---

## Comparison

| Clock Type | Causality | Wall Time | Goes Backward | Use Case |
|-----------|-----------|-----------|---------------|----------|
| Physical (NTP) | No | Yes | Yes (NTP) | Human-readable time |
| Lamport | Yes | No | Never | Total ordering |
| Vector clock | Yes (precise) | No | Never | Conflict detection |
| **HLC** | **Yes** | **Approx** | **Never** | **Distributed DB timestamps** |
| TrueTime (Spanner) | Yes | Yes (bounded) | Never | Linearizability with commit wait |

---

## Used In
- **CockroachDB** — primary clock for all transaction timestamps
- **YugabyteDB** — same approach
- Academic basis: Kulkarni et al. "Logical Physical Clocks" (2014)
