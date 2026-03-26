# Distributed Systems Fundamentals

## Why Distributed Systems?

- Problems exceed single-machine capacity (storage, compute)
- Mid-range commodity hardware = optimal cost-value
- High-end hardware advantage diminishes as cluster size grows

### Three Scalability Dimensions
| Dimension | Goal |
|-----------|------|
| Size | Adding nodes linearly speeds up the system; larger data shouldn't increase latency |
| Geographic | Multiple DCs reduce response time; manage cross-DC latency |
| Administrative | Adding nodes doesn't increase admin cost (admin-to-machine ratio) |

---

## Availability

```
Availability = uptime / (uptime + downtime)
```

| Nines | Annual Downtime |
|-------|----------------|
| 90% (1-nine) | >1 month |
| 99% (2-nines) | <4 days |
| 99.9% (3-nines) | <9 hours |
| 99.99% (4-nines) | <1 hour |
| 99.999% (5-nines) | ~5 min |
| 99.9999% (6-nines) | ~31 sec |

- Non-redundant systems: availability = component availability
- Redundant systems: tolerate partial failures → higher availability

---

## Fault Tolerance

- **Fault tolerance**: system behaves in a well-defined manner when faults occur
- Only *anticipated* faults can be tolerated — unanticipated ones cannot
- **Partial failure**: parts of the system fail while others work (unique to distributed systems)

### Failure Types
| Type | Description | Realistic? |
|------|-------------|-----------|
| Crash-stop | Node crashes, gone forever | Simple model |
| Crash-recovery | Node crashes, may come back | **Most realistic** |
| Byzantine | Node lies or sends malicious data | Rare in datacenters |

---

## Performance Trade-offs

- **Latency**: hits physical limits (speed of light, hardware minimums) — not just financial
- **Throughput vs latency**: larger batches → higher throughput, longer individual response time
- **Minimum latency**: depends on query nature + physical distance between data sources

---

## Design Techniques: Partition and Replicate

### Partitioning
- Split data across nodes
- Improves performance: limits data examined, co-locates related data
- Improves availability: partitions fail independently
- Highly application-specific

### Replication
- Copy identical data to multiple machines
- Improves performance: more bandwidth + computing power
- Improves availability: more copies = more nodes must fail before data loss
- **Causes consistency problems** — independent copies must stay synchronized

> "To replication! The cause of, and solution to, all of life's problems." — Mixu

---

## System Models

A system model specifies: node capabilities, failure modes, network properties, timing assumptions.

### Timing Models
| Model | Description | Realistic? |
|-------|-------------|-----------|
| Synchronous | Bounded delays, lock-step execution, accurate clocks | No |
| **Partially synchronous** | Usually bounded, occasional spikes | **Yes** |
| Asynchronous | No timing assumptions, unbounded delays | Too restrictive |

### Network Properties
- Nodes communicate over FIFO-ordered links (typically)
- Messages can be: lost, reordered, delayed arbitrarily
- **Network partition**: nodes alive but can't communicate with each other

---

## CAP Theorem

Cannot simultaneously guarantee all three:
| Property | Meaning |
|----------|---------|
| **C**onsistency | All nodes see identical data simultaneously |
| **A**vailability | Node failures don't stop the system serving requests |
| **P**artition tolerance | System continues despite message loss between nodes |

| Choice | Sacrifice | Example |
|--------|-----------|---------|
| CP | Availability during partitions | HBase, ZooKeeper |
| AP | Strong consistency | Cassandra, DynamoDB |
| CA | Partition tolerance (not viable in practice) | Single-node RDBMS |

**Key nuances:**
- Partitions *will* happen — P is not optional in practice
- Linearizability is always slow, even without network faults
- "Consistency" in CAP ≠ ACID consistency ≠ other guarantees

---

## FLP Impossibility

> "In an asynchronous system with reliable networks and crash-only failures, there is no deterministic algorithm for consensus — even if at most one process may fail."
> — Fischer, Lynch, Paterson (1985)

- Forces trade-offs between **safety** (nothing bad happens) and **liveness** (something good eventually happens)
- Practical systems work around this with timeouts + probabilistic assumptions

---

## Safety vs. Liveness

| Property | Definition | Example |
|----------|-----------|---------|
| Safety | "Nothing bad happens" — must always hold | Uniqueness constraint, no dirty reads |
| Liveness | "Something good eventually happens" — caveats allowed | Availability, eventual consistency |

Real systems: guarantee safety always, liveness under partial failures only.
