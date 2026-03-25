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
