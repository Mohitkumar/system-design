# Unique ID Generation in Distributed Systems

**Requirements:** 64-bit numeric IDs, globally unique, ~1 billion IDs/day, roughly sortable by time.

---

## 1. UUID (v4)

128-bit random identifier.

```
550e8400-e29b-41d4-a716-446655440000
```

| | |
|---|---|
| Capacity | 10^38 IDs |
| **Cons** | 128-bit — hurts DB index performance (doubles B-tree node size) |
| | Non-numeric, non-sortable |
| | No causality — you can't tell which ID came first |
| | Random inserts cause B-tree fragmentation (page splits) |

---

## 2. Auto-increment with Multiple DB Servers

Each server generates IDs using `AUTO_INCREMENT` but with an offset per server.

```
Server 1: 1, 3, 5, 7 ...   (start=1, increment=2)
Server 2: 2, 4, 6, 8 ...   (start=2, increment=2)
```

| | |
|---|---|
| **Cons** | Adding servers requires resharding the increment step across all existing servers |
| | A failed server creates gaps or, if replaced carelessly, duplicate IDs |
| | IDs don't sort reliably across servers (server 2 can generate a lower ID after server 1) |
| | Hard upper bound on throughput per shard |

---

## 3. Twitter Snowflake

64-bit ID split into fixed fields:

```
| 1 bit  | 41 bits          | 10 bits    | 12 bits  |
| unused | ms since epoch   | worker ID  | sequence |
```

- **41-bit timestamp** → 2^41 ms ≈ 69 years from a chosen epoch. After that, the epoch must be reset or the bit layout changed.
- **10-bit worker ID** → up to 1024 workers (can split 5 bits datacenter + 5 bits machine).
- **12-bit sequence** → 4096 IDs per millisecond per worker → ~4M IDs/sec per worker.

IDs are **time-sortable** and numeric. Workers generate IDs independently with no coordination.

**Used by:** Twitter, Discord (modified), Instagram (Postgres-based variant).

---

## 4. Sonyflake (Sony's Snowflake variant)

Same idea, different bit split — optimized for more machines at lower throughput:

```
| 39 bits       | 8 bits   | 16 bits  |
| ms since 2014 | sequence | machine  |
```

- Runs for 174 years (39-bit timestamp in 10ms units).
- Supports up to 2^16 = 65536 machines — better for large clusters.
- Lower sequence (8 bits) → 256 IDs per 10ms per machine (much lower throughput).

---

## 5. ULID (Universally Unique Lexicographically Sortable Identifier)

128-bit like UUID but sortable:

```
| 48 bits (ms timestamp) | 80 bits (random) |
01ARZ3NDEKTSV4RRFFQ69G5FAV
```

- Encodes as 26-character base32 string (case-insensitive, URL-safe).
- Lexicographic sort = chronological sort.
- No coordination needed — randomness in lower 80 bits makes collisions negligible.
- **Cons:** Still 128-bit (same index cost as UUID); string representation adds overhead vs pure integer.

---

## 6. Ticket Server (Flickr-style)

A dedicated centralized DB acts as the sole ID issuer:

```sql
REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

- Simple, numeric, monotonically increasing.
- **Cons:** Single point of failure (mitigated by running two ticket servers with odd/even split, same tradeoff as §2). Becomes a bottleneck at very high throughput. Network round-trip per ID.

**Used by:** Flickr.

---

## 7. Vector Clock as ID (theoretical)

Replace the epoch in Snowflake with a vector clock counter:

```
| 1 bit  | 53 bits           | 10 bits   |
| unused | vector clock val  | worker ID |
```

- Captures **causality** — you can tell which ID causally preceded another.
- **Cons:** The full vector (one counter per node) grows with cluster size and must be transmitted over the network. Doesn't compress into 64 bits cleanly for large clusters. Overkill if you only need ordering, not causality.

---

## Comparison

| Strategy | Bits | Sortable | Causality | Coordination | Throughput |
|---|---|---|---|---|---|
| UUID v4 | 128 | No | No | None | Very high |
| Multi-DB auto-inc | 64 | Partial | No | Schema config | Medium |
| Snowflake | 64 | Yes (time) | No | None | ~4M/sec/worker |
| Sonyflake | 64 | Yes (time) | No | None | ~25/sec/worker |
| ULID | 128 | Yes (lex) | No | None | Very high |
| Ticket server | 64 | Yes | No | Centralized | Low–Medium |
| Vector clock ID | 64 | Yes | Yes | Gossip/sync | Medium |

---

# Unique ID Generation — System Design (Snowflake)

---

## Requirements

**Functional**
- Generate globally unique 64-bit numeric IDs
- IDs must be sortable by generation time
- `generateId()` → `uint64`

**Non-Functional**

| Requirement | Target |
|---|---|
| Throughput | 1B IDs/day ≈ ~12K IDs/sec peak (assume 3× headroom → 36K/sec) |
| Latency | < 2ms p99 |
| Availability | 99.99% |
| No coordination | Workers generate IDs independently — no DB, no locks |
| Monotonic within worker | IDs from the same worker are always increasing |

---

## Capacity Estimation

- 1B IDs/day ÷ 86,400 sec = **~11,574 IDs/sec average**
- **80/20 rule:** 80% of requests arrive in 20% of the day (peak ~5 hours) → peak = (1B × 0.8) / (0.2 × 86,400) ≈ **~46K IDs/sec**
- Each Snowflake worker: 4096 IDs/ms = **4M IDs/sec**
- Workers needed: 46,000 / 4,000,000 ≈ **1 worker is enough**; run **5–10 workers** for HA and geo-distribution

**IOPS:**
- ID generation is **pure in-memory** (bit manipulation on timestamp + counter)
- Zero disk IOPS per ID generated
- Worker ID assignment: 1 etcd read on startup (one-time)
- Network: 46k req/sec × ~50 bytes/response = **~2.3 MB/s** — trivial

**Server count:**
| Component | Count | Sizing | Reason |
|-----------|-------|--------|--------|
| ID generator workers | 5–10 | 2 cores, 4 GB RAM | Stateless; CPU-bound not I/O-bound |
| Load balancer | 2 | L7 (round-robin) | HA for worker fleet |
| etcd (worker ID lease) | 3 | 4 cores, 8 GB | Quorum for worker ID assignment |

- Each worker: single goroutine/thread handles sequence; multi-thread needs lock → keep single-threaded
- At 46k req/sec per worker, 1 CPU core at ~1M simple ops/sec → **CPU is never the bottleneck**
- Bottleneck in practice: **clock resolution** (1ms granularity → 4096 max/ms)
  - At 4M IDs/sec ceiling per worker, 10 workers → **40M IDs/sec** max throughput

---

## Bit Layout

```
 63        22 21      12 11       0
 ┌──────────┬──────────┬──────────┐
 │ 41 bits  │ 10 bits  │ 12 bits  │
 │timestamp │ workerID │ sequence │
 └──────────┴──────────┴──────────┘
  (sign bit = 0, always positive)
```

| Field | Bits | Max value | Notes |
|---|---|---|---|
| Sign | 1 | 0 | Always 0; keeps IDs positive |
| Timestamp | 41 | 2^41 ms = 69 years | ms since custom epoch (e.g. 2024-01-01) |
| Worker ID | 10 | 1024 workers | Split: 5 bits datacenter + 5 bits machine |
| Sequence | 12 | 4096 per ms | Resets to 0 each millisecond |

**ID construction:**
```
id = (timestamp << 22) | (workerID << 12) | sequence
```

---

## High-Level Architecture

```
                     ┌─────────────────────┐
                     │   Config Service     │  (ZooKeeper / etcd)
                     │  assigns worker IDs  │
                     └────────┬────────────┘
                              │ worker ID on startup
              ┌───────────────┼──────────────┐
              ▼               ▼              ▼
        [Worker 0]      [Worker 1]     [Worker N]
        DC1 / M1        DC1 / M2       DC2 / M1
              │               │
         ────────────────────────────
              │  HTTP/gRPC /generateId()
              ▼
         [ Clients ]
```

- Workers are **stateless** except for their worker ID and last-used timestamp.
- Worker ID is leased from the config service at startup and released on shutdown.
- No inter-worker communication needed during ID generation.

---

## Worker Internals

```
state:
  workerID    = assigned at startup
  lastMs      = last millisecond a batch was issued
  sequence    = counter within current ms (0–4095)

generateId():
  now = currentTimeMs()

  if now == lastMs:
    sequence++
    if sequence > 4095:          // exhausted 4096 slots in this ms
      wait until next ms         // spin or sleep 1ms
      now = currentTimeMs()
      sequence = 0
  else:
    sequence = 0

  if now < lastMs:               // clock moved backward
    throw ClockBackwardException

  lastMs = now
  return (now - EPOCH) << 22 | workerID << 12 | sequence
```

---

## Clock Skew & Clock Backward Problem

Snowflake assumes the system clock only moves forward. NTP corrections can move the clock backward.

| Scenario | Problem | Fix |
|---|---|---|
| Small backward jump (< 10ms) | Duplicate timestamp → possible duplicate ID | Spin-wait until `now >= lastMs` |
| Large backward jump | Long stall blocking ID generation | Alert + pause worker; let ops investigate |
| Clock drift across workers | IDs from different workers not globally monotonic | Acceptable — only monotonic *within* a worker |

---

## Worker ID Assignment

Workers must have unique IDs. Two approaches:

| Approach | How | Tradeoff |
|---|---|---|
| **Zookeeper / etcd lease** | Worker claims a node on startup; releases on shutdown. Ephemeral node expires if worker crashes. | Correct; requires ZK dependency |
| **Static config** | Each host is pre-assigned an ID via config (e.g. k8s pod env var) | Simple; risk of misconfiguration → duplicate IDs |
| **DB row lock** | Worker inserts a row claiming an ID; deletes on exit | Simple; DB becomes a dependency |

**Recommended:** etcd lease — atomic claim, auto-release on crash.

---

## Availability & Failure Handling

| Failure | Impact | Mitigation |
|---|---|---|
| Single worker crash | That worker's capacity lost | Run N workers; clients round-robin |
| Config service down | New workers can't get IDs | Workers cache their ID; existing workers unaffected |
| Clock moves backward | Worker stalls | Spin-wait (small) or alert (large); do NOT generate |
| Worker ID exhausted (all 1024 used) | No new workers | Reclaim IDs of dead workers via lease expiry |

---

## Scalability

- At 4096 IDs/ms, a **single worker** handles ~4M IDs/sec — far beyond most systems.
- Scale out workers for **geo-distribution** (low latency to regional clients) and **HA**, not throughput.
- With 10-bit worker field: max **1024 workers** globally. If more needed, sacrifice sequence bits (10 bits worker → 11 bits = 2048 workers, 11 bits sequence → 2048 IDs/ms).

---

## Summary Cheatsheet

| Decision | Choice | Reason |
|---|---|---|
| ID size | 64-bit integer | DB index friendly, fits in most languages natively |
| Time source | System clock (ms) | No coordination; NTP keeps it accurate enough |
| Worker coordination | etcd lease for worker ID | Unique assignment without central bottleneck at generation time |
| Clock backward | Spin-wait / alert | Safety over availability on clock issues |
| Throughput | 4096/ms per worker | Exceeds any realistic single-service requirement |
| Epoch | Custom (e.g. 2024-01-01) | Maximizes the 69-year lifespan |

---

## Interview Scenarios

### "Clock skew — NTP jumps time backward by 500ms"
- Snowflake: spin-wait until `currentTime >= lastTimestamp`; do NOT generate IDs during backward clock
- For large skew (>1s): alert on-call + stop the worker; restart after clock stabilizes
- Detection: monitor `clock_skew_ms` metric per worker; set alert at >10ms
- Prevention: use `CLOCK_MONOTONIC` for sequence; sync with `CLOCK_REALTIME` only for epoch bits

### "Worker ID exhaustion — all 1024 worker IDs in use"
- Root cause: old workers not releasing IDs (crash without cleanup, or leaked leases)
- Fix: short TTL leases (60s) in etcd; dead worker's ID reclaimed automatically on lease expiry
- Monitoring: alert when `available_worker_ids < 50`; scale out ID tier before exhaustion
- If need more workers: trade 1 sequence bit for 1 worker bit → 2048 workers, 2048 IDs/ms (still ~2M IDs/sec per worker)

### "Need IDs that sort by insertion order (user wants to paginate by ID)"
- Snowflake IDs are already time-sorted: higher ID = later creation time (ms granularity)
- For sub-ms sort within same tick: sequence number preserves order within a worker, but cross-worker order within same ms is not deterministic
- If strict total order needed: use a distributed sequence (Redis INCR) — but this creates a bottleneck
- Alternative: ULID (Universally Unique Lexicographically Sortable Identifier) — 48-bit time + 80-bit random, lexicographically sortable

### "ID must be globally unique across regions"
- Snowflake worker IDs must be region-scoped: assign datacenter_id (5 bits) + worker_id (5 bits) — separates ID spaces by region
- Each region's workers generate IDs independently; no cross-region coordination needed
- Risk: if same worker_id is assigned in two regions → duplicate IDs; prevent via centralized worker ID registry (global etcd)

### "Constraint: IDs must be unpredictable (security — can't enumerate resources)"
- Snowflake IDs are guessable (timestamp + worker + sequence) — bad for security-sensitive resources
- Use UUID v4 (random 122 bits) for resource IDs that must not be enumerable
- Hybrid: generate Snowflake for internal primary key; expose UUID v4 as external API identifier
- Or: HMAC-sign the Snowflake ID with a secret → opaque external ID that maps back internally

### "Database becomes the bottleneck on ID generation"
- Auto-increment DB sequences: single bottleneck, ~10k IDs/sec ceiling
- Fix: batch allocation — worker requests a range (e.g., 1000 IDs) from DB; serves locally without DB per request
- Range size tradeoff: large range = more IDs lost on crash; small range = more DB round trips
- Snowflake approach eliminates DB entirely — preferred for high-throughput systems
