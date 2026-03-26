# Time and Ordering in Distributed Systems

## Why Order Matters

- Single-machine programs have a natural **total order** (sequential execution)
- Distributed systems have **partial order** — some events are concurrent and incomparable
- Transforming partial → total order requires communication, waiting, and restricts parallelism

### Total vs. Partial Order
| Model | Description |
|-------|-------------|
| Total order | Every element comparable — single-threaded execution |
| Partial order | Some elements incomparable — concurrent operations on different nodes |

---

## Three Time Models

| Model | How | Limitation |
|-------|-----|-----------|
| Global Clock | Perfect sync — total order without communication | Only possible to limited accuracy (hardware + physics) |
| Local Clock | Each machine has own clock | Only orders events *within* a single node |
| No Clock (Logical Time) | Counters + communication track causality | Must communicate to establish order |

---

## Physical Clocks

### Time-of-Day Clock
- Returns current date/time
- Synchronized with NTP
- **Can jump backward** (NTP correction)
- **Do NOT use for measuring elapsed time or ordering events across nodes**

### Monotonic Clock
- Always moves forward — suitable for measuring durations (timeouts, intervals)
- Absolute value is meaningless — only differences matter
- NTP may adjust frequency (slew), not position

### Clock Accuracy in Practice
- Best practical accuracy with NTP: **tens of milliseconds**
- GPS receivers + PTP: microsecond accuracy
- **Never rely on wall-clock timestamps for event ordering across distributed nodes**

---

## Lamport Clocks (Logical Clocks)

Simple counter-based ordering — forces a **total order**.

### Rules
1. Increment counter on every local event
2. Attach counter to every outgoing message
3. On receive: `counter = max(local, received) + 1`

### Properties
- `timestamp(a) < timestamp(b)` → a *may* have preceded b (or they are concurrent)
- Cannot distinguish concurrent from causally dependent events
- **Creates total order by breaking ties with node ID**

### Example
```
Client A write: timestamp=10, nodeID=1
Client B write: timestamp=10, nodeID=2
→ Tie-break by nodeID → A wins; B's write silently discarded
```

### Use Cases (when agreement > history)
- Distributed locks — just need one winner
- Total-order multicast — all nodes process in same sequence
- Leader election — pick one node without debate

### Real Systems
| System | Usage |
|--------|-------|
| ZooKeeper | `zxid` — Lamport-style monotonic counter for total transaction ordering |
| Google Chubby | Sequencer tokens for lock acquisitions |
| Kafka | Log offsets — monotonic, total order within a partition |

---

## Vector Clocks

Array of counters — one per node. Provides accurate **causality tracking**.

### Rules
1. Increment own slot on every local event
2. Attach full vector to outgoing messages
3. On receive: merge element-wise max, then increment own slot

### Example
```
Node A writes: [A:1, B:0]
Node B writes: [A:0, B:1]
→ Neither dominates → concurrent writes → conflict detected
```

### Comparison Rules
- `V1 < V2` if every element of V1 ≤ V2 (and at least one strictly less) → V1 happened before V2
- Neither dominates → **concurrent** (potential conflict)

### Properties
- Correctly identifies concurrent vs. causally dependent operations
- Cost scales linearly with node count
- Used as a **conflict detector**, not a conflict resolver

### Real Systems
| System | Usage |
|--------|-------|
| Amazon DynamoDB | Tracks causality between replica versions; exposes conflicts to caller |
| Riak | Stores conflicting versions as siblings; client merges |
| Voldemort (LinkedIn) | Detects stale reads, surfaces conflicts |

---

## Failure Detectors

Abstract timing assumptions using heartbeats + timeouts.

### The Problem
In asynchronous systems, **you cannot distinguish a failed node from a slow node** — both look like "no response."

### Chandra-Toueg Properties
| Property | Meaning |
|----------|---------|
| Completeness | Crashed processes are *eventually* suspected |
| Accuracy | Non-faulty processes are never falsely suspected (hard to guarantee) |

### Key Insight
- Even **weak** failure detectors enable solving consensus in async systems
- Timeout value is critical (see DDIA ch8)

### Timeout Trade-offs
| Timeout | Risk |
|---------|------|
| Too short | False positives — declare live node dead → duplicate actions, cascading overload |
| Too long | Slow fault detection |
| Theoretical optimal | `2d + r` (max packet delay + processing time) — not bounded in practice |

**Best practice**: continuously measure round-trip times, set timeouts experimentally.

---

## Process Pauses

A node cannot detect its own pause. Causes:
- GC stop-the-world (can be seconds)
- VM suspension / live migration
- OS context switches, disk I/O waits
- Swapping to disk

### Consequence
A node that wakes up after a pause may act on **stale assumptions** — e.g., still believe it holds a lease that expired.

**Fix**: Fencing tokens — monotonically increasing token with each lock grant; storage rejects writes with token ≤ last seen.

---

## Summary: Which Clock/Mechanism to Use

| Need | Use |
|------|-----|
| Measure elapsed time (local) | Monotonic clock |
| Current time for humans | Time-of-day clock |
| Total ordering (agreement, no conflict history needed) | Lamport timestamps |
| Detect concurrent writes / causality | Vector clocks |
| Detect node failures | Failure detector + heartbeats |
| Prevent stale-lock writes | Fencing tokens |
