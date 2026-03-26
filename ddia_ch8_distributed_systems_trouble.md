# DDIA Chapter 8: The Trouble with Distributed Systems

## Theory

### Core Principle
> Everything that can go wrong, will go wrong — design assuming partial failures.

- **Single computer**: deterministic — fully works or fully crashes
- **Distributed system**: nondeterministic — parts fail while others work fine

---

## Unreliable Networks

- Shared-nothing commodity hardware connected by async packet networks
- Possible outcomes for any request: lost, queued, slow, receiver crashed, response lost
- **Only practical solution: timeouts** — but what is the right timeout?

### Timeout Trade-offs
| Timeout | Risk |
|---------|------|
| Too short | False positives — declare alive node dead → duplicate actions, cascading overload |
| Too long | Slow fault detection |
| Optimal theoretical | `2d + r` (max packet delay + processing time) — not bounded in practice |

- **Best practice**: measure round-trip times continuously, set timeouts experimentally
- **UDP** preferred over TCP when delayed data is worthless (video, voice)

---

## Unreliable Clocks

| Clock Type | Use | Notes |
|-----------|-----|-------|
| Time-of-day | Current date/time | Synced via NTP; can jump backward; **not for durations** |
| Monotonic | Measuring elapsed time | Always moves forward; meaningless absolute value |

- NTP accuracy: **tens of milliseconds** in practice (worse on congested networks)
- Better accuracy: GPS receivers, PTP (Precision Time Protocol), careful monitoring
- **Never use wall-clock timestamps for event ordering** across nodes — use logical clocks

### Clock Hazards
- Threads can pause (GC, page faults, VM migration) — node can't detect its own pause
- A node that wakes up may act on stale assumptions (e.g. still holding a lease)
- Represent time as an interval (lower, upper bound), not a point

---

## Process Pauses
- GC stop-the-world (seconds)
- VM suspension (live migration)
- OS context switches, disk I/O waits
- **A node cannot tell the difference between "just paused" and "just crashed"**

---

## Knowledge, Truth, and Lies

### A Node Cannot Know Anything Alone
- Must rely on quorum decisions
- Even if a node believes it has a lock, the lock may have expired

### Fencing Tokens
- Monotonically increasing token issued with each lock grant
- Storage service rejects writes with token ≤ last seen token
- Prevents stale lock holder from corrupting data

### Byzantine Faults
| Fault Type | Description | Practical? |
|-----------|-------------|-----------|
| Crash-stop | Node fails and never returns | Yes |
| Crash-recovery | Node can crash and recover | Yes (most realistic) |
| Byzantine | Node lies, sends incorrect/malicious data | Rarely (peer-to-peer, aerospace) |

- Datacenter nodes: assume **crash-recovery**, not Byzantine
- Mitigate weak lying: checksums (TCP + app level), input validation, sanity checks

---

## System Models

### Timing Models
| Model | Assumption | Realistic? |
|-------|-----------|-----------|
| Synchronous | Bounded delays | No |
| Partially synchronous | Usually bounded, occasional spikes | **Yes** |
| Asynchronous | No timing assumptions | Too restrictive |

### Node Failure Models
| Model | Assumption |
|-------|-----------|
| Crash-stop | Node crashes, gone forever |
| Crash-recovery | Node crashes, may come back |
| Byzantine | Node can do anything |

**Best real-world model: partially synchronous + crash-recovery**

---

## Safety vs. Liveness
| Type | Meaning | Example |
|------|---------|---------|
| Safety | "Nothing bad happens" — must always hold | Uniqueness, no dirty reads |
| Liveness | "Something good eventually happens" — caveats allowed | Availability, eventual consistency |

---

## System Design Implications
- Always design for network partitions and partial failures
- Never rely on clocks for ordering — use logical clocks / version vectors
- Use fencing tokens when handing out locks/leases
- Build in timeouts + retries with idempotency (to handle duplicate actions safely)
- Test with chaos engineering (inject faults artificially)
