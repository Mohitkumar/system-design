# Amazon DynamoDB — System Design

## What Is It
Fully managed, serverless, key-value + document NoSQL database.
- Single-digit millisecond latency at any scale
- Auto-scaling, multi-AZ by default, multi-region optional
- Based on Amazon Dynamo paper (2007) — but significantly evolved

---

## Functional Requirements
- GetItem, PutItem, UpdateItem, DeleteItem (single key, strongly or eventually consistent)
- BatchGetItem, BatchWriteItem (up to 25 items)
- Query (by partition key, optional sort key range)
- Scan (full table — avoid in production)
- Transactions: TransactWriteItems, TransactGetItems (up to 25 items, 2PC internally)
- Global Secondary Index (GSI), Local Secondary Index (LSI)
- Streams (CDC — item-level change capture)
- TTL (auto-expire items)
- Global Tables (multi-region active-active replication)

## Non-Functional Requirements
- **Latency**: single-digit ms p99 reads/writes
- **Availability**: 99.999% (Global Tables), 99.99% (single-region)
- **Durability**: multi-AZ replication (3 AZs)
- **Scale**: no practical limit — auto-partitions as data/traffic grows
- **Consistency**: eventual (default reads) or strong (consistent reads, 2× cost)
- Serverless — no cluster to manage, no capacity pre-planning with on-demand mode

---

## Capacity Estimation

```
DAU:              50M users
Reads/user/day:   100 → 50M × 100 / 86,400 ≈ 58k RCU avg, peak ≈ 175k RCU
Writes/user/day:  10  → 50M × 10 / 86,400  ≈ 5.8k WCU avg, peak ≈ 17k WCU

Item size: 1 KB avg
RCU pricing:  1 RCU = 1 strongly consistent read of 4 KB (or 2 eventually consistent)
WCU pricing:  1 WCU = 1 write of 1 KB

Storage:
  50M users × 10 items × 1 KB = 500 GB
  Growth: 5.8k writes/s × 86,400s × 1 KB ≈ 500 GB/day new data
  5yr:    500 GB/day × 1,825 = ~912 TB

IOPS:
  Write: 17k WCU/s peak; each WCU = 1 write to leader + replicate to 2 followers
         Leader write: 17k random writes/s (B-tree in-place updates)
         WAL: 17k sequential writes/s (for crash recovery)
         Per-partition: 1,000 WCU/s max → 17 partitions minimum for write load
  Read:  175k RCU/s peak (eventually consistent hits any replica)
         B-tree read: ~3–4 random IOs per cache miss; assume 5% miss = 8.75k random IOPS
         Per-partition: 3,000 RCU/s max → 58 partitions minimum for read load
  NVMe SSD per node: 500k IOPS → single node handles both read+write IOPS easily
  Bottleneck: throughput capacity (RCU/WCU limits) not raw IOPS

Server count (DynamoDB is fully managed — for self-hosted equivalent):
  Storage nodes: 912 TB / 10 TB per node = ~92 nodes; add replication = ~276 nodes
  Partition manager: 3–5 nodes (metadata routing)
  In practice (AWS DynamoDB): thousands of storage nodes globally per cell
```

---

## Data Model

### Key Structure
```
Table = collection of items (rows)

Primary Key — two options:
  1. Partition key only (simple PK):
     PK = user_id → uniquely identifies an item
     Use: one item per entity (user profile, product)

  2. Partition key + Sort key (composite PK):
     PK = user_id,  SK = timestamp#event_type
     → all items with same PK stored together, sorted by SK
     → enables: GetItem(PK), Query(PK, SK between X and Y)
     Use: one-to-many (user → orders, chat messages, activity log)
```

### Item Structure
```json
{
  "user_id":    "u_12345",          ← Partition Key
  "timestamp":  "2024-01-15T10:00", ← Sort Key
  "event_type": "purchase",
  "amount":     49.99,
  "items":      ["item_a", "item_b"],
  "metadata":   { "source": "mobile" },
  "ttl":        1705320000           ← Unix epoch; auto-deleted after this time
}
```

- No schema enforcement (except PK must exist)
- Max item size: **400 KB**
- Attribute types: String, Number, Binary, Boolean, Null, List, Map, StringSet, NumberSet, BinarySet

---

## Architecture

```
Client
  │
  ├── HTTPS request (SigV4 auth)
  ▼
┌─────────────────────────────────────────────────────────┐
│                   DynamoDB Service                       │
│                                                          │
│  ┌──────────────┐    ┌────────────────┐                 │
│  │  Request     │    │   Metadata     │                 │
│  │  Router      │───►│   Service      │                 │
│  │  (auth,      │    │   (partition   │                 │
│  │   routing)   │    │    map, table  │                 │
│  └──────────────┘    │    config)     │                 │
│         │            └────────────────┘                 │
│         │                    │                          │
│         ▼                    ▼                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │            Storage Nodes (B-tree per partition) │    │
│  │                                                 │    │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐     │    │
│  │  │ Leader  │───►│Follower │    │Follower │     │    │
│  │  │  AZ-1   │    │  AZ-2   │    │  AZ-3   │     │    │
│  │  └─────────┘    └─────────┘    └─────────┘     │    │
│  │   (writes + strong reads)   (eventually consistent)  │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  DynamoDB Streams  │  GSI Updater  │  TTL Worker │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Partitioning

### Partition Key → Shard Mapping
```
Internal hash = MD5(partition_key_value)
→ maps to one of N partitions (automatically managed)

Each partition:
  - Max 10 GB data
  - Max 3,000 RCU + 1,000 WCU
  → DynamoDB auto-splits when either limit approached

Partition count = max(ceil(storage/10GB), ceil(WCU/1000), ceil(RCU/3000))
```

### Adaptive Capacity (Burst Handling)
- Each partition has a throughput budget
- **Burst capacity**: unused capacity pooled; short spikes absorbed (up to 300s of unused)
- **Adaptive capacity (2019)**: DynamoDB detects hot partitions and automatically increases their throughput budget
- **Instant adaptive capacity**: hot item partitions get proportionally higher limits automatically

### Hot Partition Problem
```
BAD:  PK = "status" (values: "active", "inactive") → 2 partitions, all traffic on "active"
BAD:  PK = date (e.g. "2024-01-15") → all today's writes to one partition
GOOD: PK = user_id → evenly distributed by user hash
GOOD: PK = device_id#shard_suffix (write sharding: append random 0-9 suffix)
      → 10 partitions per logical key; read from all 10 and merge
```

---

## Consistency Models

| Mode | How | Latency | Cost | Use When |
|------|-----|---------|------|----------|
| **Eventually consistent read** | Returns from any replica | Lowest | 0.5 RCU per 4KB | Default — stale reads tolerable |
| **Strongly consistent read** | Returns from leader only | Slightly higher | 1 RCU per 4KB | Must see latest write |
| **Transactional read** | 2PC across items | Higher | 2 RCU per 4KB | Cross-item atomicity |

```
// Strongly consistent read
GetItem({
  TableName: "Orders",
  Key: { user_id: "u_123", order_id: "o_456" },
  ConsistentRead: true   ← goes to leader replica
})
```

---

## Indexes

### Local Secondary Index (LSI)
- Same partition key as table, **different sort key**
- Must be created at table creation time (cannot add later)
- Shares RCU/WCU with base table
- **Strongly consistent reads supported**
- Max 5 LSIs per table
- Use: query by same entity with different sort attribute

```
Table: PK=user_id, SK=order_date
LSI:   PK=user_id, SK=order_total  ← query "most expensive orders for user X"
```

### Global Secondary Index (GSI)
- **Different partition key** (and optional sort key) from base table
- Can be created/deleted any time
- **Own provisioned capacity** (separate RCU/WCU from table)
- **Eventually consistent only** (replicated async from base table)
- Max 20 GSIs per table
- Use: query by non-primary-key attribute

```
Table: PK=user_id,  SK=order_id
GSI:   PK=product_id, SK=order_date  ← "all orders for product X, sorted by date"
```

### GSI Write Behavior
```
Write to base table → DynamoDB async replicates to GSI
→ GSI reads may lag behind base table by milliseconds
→ GSI has own WCU — if GSI WCU exhausted: writes to base table throttled too
→ Keep GSI WCU >= base table WCU on write-heavy attributes
```

### Sparse Index
```
GSI only includes items where the indexed attribute EXISTS
Items without the attribute are not indexed → saves cost

Example:
  Base table: all users (most have no "premium_since" attribute)
  GSI on "premium_since": only premium users indexed
  → Query GSI to get all premium users cheaply
```

---

## Transactions

DynamoDB supports ACID transactions across multiple items/tables (same region).

### TransactWriteItems
```
Up to 25 items, all-or-nothing:
  TransactWriteItems([
    { Put:    { TableName: "Inventory", Item: {...}, ConditionExpression: "stock > 0" }},
    { Update: { TableName: "Orders",    Key: {...},  UpdateExpression: "SET status = :s" }},
    { Delete: { TableName: "Cart",      Key: {...} }}
  ])
```

### Internal Implementation (2PC)
```
Phase 1 (Prepare):
  - Lock all items (write conflict markers)
  - Validate all condition expressions
  - If any fails → abort, release all locks

Phase 2 (Commit):
  - Apply all writes atomically
  - Remove conflict markers
  - Async replicate to GSIs + Streams

Cost: 2 WCU per item write (vs 1 WCU for normal write)
```

### ConditionExpression (Optimistic Locking)
```
// Only update if version matches (optimistic locking)
UpdateItem({
  Key: { id: "item_1" },
  UpdateExpression: "SET #v = :new_v, data = :data",
  ConditionExpression: "#v = :expected_v",
  ExpressionAttributeValues: {
    ":new_v": 6, ":data": "new_data", ":expected_v": 5
  }
})
// If version != 5 → ConditionalCheckFailedException → retry
```

---

## DynamoDB Streams

Item-level change data capture (CDC).

```
Every write to table → appended to stream (ordered per partition key)
Stream records:
  - NEW_IMAGE:     item after change
  - OLD_IMAGE:     item before change
  - NEW_AND_OLD:   both
  - KEYS_ONLY:     only PK/SK

Retention: 24 hours
Consumers: Lambda (event source mapping), Kinesis Data Streams adapter

Use cases:
  - Replicate to Elasticsearch for search
  - Invalidate cache on item change
  - Fan-out to downstream services
  - Audit log
  - Cross-region custom replication
```

### Lambda + Streams Pattern
```
DynamoDB Table → Stream → Lambda (batch size=100, bisect on error)
                               │
                        ┌──────┴──────┐
                    Elasticsearch   Redis cache
                    (update index)  (invalidate key)
```

---

## Global Tables (Multi-Region Active-Active)

```
Region A (us-east-1)  ←──async replication──► Region B (eu-west-1)
     │                                              │
  Writes OK                                     Writes OK
  (any region)                                  (any region)

Conflict resolution: last-writer-wins by timestamp
→ near-simultaneous writes to same item → one silently wins
→ avoid: application-level partitioning (region A owns users A-M, region B owns N-Z)
```

- Replication lag: typically < 1 second globally
- **Availability**: 99.999% (five nines)
- All regions share same table schema and indexes
- **RPO ≈ 0, RTO ≈ 0** for regional failure (other regions continue serving)

---

## TTL (Auto Expiration)

```
Item: { user_id: "u_123", session_token: "abc", ttl: 1705320000 }

TTL attribute: epoch seconds (must be Number type)
DynamoDB background process:
  - Scans for items with ttl < current_time
  - Deletes expired items within ~48 hours of expiry
  - Deletions appear in Streams (can trigger Lambda cleanup)
  - No WCU charged for TTL deletes
```

Use: sessions, OTP codes, rate limit counters, cache entries, soft deletes.

---

## Access Patterns and Table Design

### Single-Table Design (DynamoDB best practice)

Store multiple entity types in one table using generic PK/SK names.

```
Table: MyApp (PK=pk, SK=sk)

┌──────────────────────┬────────────────────────┬──────────────┐
│ pk                   │ sk                     │ other attrs  │
├──────────────────────┼────────────────────────┼──────────────┤
│ USER#u_123           │ PROFILE                │ name, email  │
│ USER#u_123           │ ORDER#o_001             │ amount, date │
│ USER#u_123           │ ORDER#o_002             │ amount, date │
│ PRODUCT#p_456        │ METADATA               │ name, price  │
│ PRODUCT#p_456        │ REVIEW#r_001            │ rating, text │
└──────────────────────┴────────────────────────┴──────────────┘

Query all orders for user:   PK=USER#u_123, SK begins_with ORDER#
Get user profile:            PK=USER#u_123, SK=PROFILE
Get product + reviews:       PK=PRODUCT#p_456 (all items)
```

**Benefits**: fewer tables, fewer RCU (fetch related items in one Query), lower latency.

### Access Pattern Design Process
```
1. List all access patterns upfront (who queries what, by which attributes)
2. Design PK/SK to satisfy most frequent patterns with Query (not Scan)
3. Add GSIs for secondary access patterns
4. Use SK prefix patterns (ORDER#, REVIEW#) to filter within partition
5. Denormalize where needed — DynamoDB has no joins
```

---

## Provisioned vs On-Demand Capacity

| | Provisioned | On-Demand |
|-|------------|-----------|
| How | Set RCU/WCU explicitly; auto-scale within bounds | Pay per request; no provisioning |
| Cost | Lower at predictable load | Higher per-request (~6-7× unit cost) |
| Throttling | Exceeds provisioned limit → throttled | Never throttled (up to 2× previous peak instantly) |
| Use when | Predictable, steady traffic | Unpredictable, spiky, new tables |
| Auto-scaling | Yes (CloudWatch → adjust RCU/WCU) | Automatic |
| Cold start | None | First surge may throttle briefly (previous peak × 2 limit) |

---

## DynamoDB Accelerator (DAX)

In-memory caching layer, fully managed, API-compatible with DynamoDB.

```
Client → DAX Cluster → DynamoDB (on cache miss)
                │
         item cache (microsecond reads)
         query cache (full result set)

Read:   GetItem hits DAX → microsecond response (vs ms from DynamoDB)
Write:  Write-through by default → updates DAX + DynamoDB synchronously
TTL:    Per-item TTL in DAX cache

Use when: read-heavy, same items accessed repeatedly, can tolerate slight staleness
Do NOT use: writes must be immediately visible, strongly consistent reads required
```

DAX latency: **microseconds** (RAM) vs DynamoDB **single-digit ms**.

---

## DynamoDB vs Other Databases

| | DynamoDB | Cassandra | MongoDB | Redis |
|-|----------|-----------|---------|-------|
| Model | KV + Document | Wide-column | Document | KV + structures |
| Managed | Fully (serverless) | Self or Astra | Atlas or self | ElastiCache or self |
| Consistency | Eventual / Strong | Tunable | Eventual / Strong | Strong (single node) |
| Transactions | Yes (25 items) | LWT (single partition) | Yes (ACID) | Yes (MULTI/EXEC) |
| Secondary indexes | GSI + LSI | Materialized views | Flexible | None native |
| Global distribution | Global Tables (AA) | Multi-DC | Atlas Global Clusters | Redis Cluster (no geo) |
| Max item size | 400 KB | Configurable | 16 MB | 512 MB value |
| Throughput | Unlimited (auto) | Manual config | Manual config | ~1M ops/s |
| Best for | AWS-native apps, serverless, spiky load | Known access patterns, time-series | Flexible schema, varied queries | Session, cache, counters |

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| PK design | Hash on high-cardinality attribute (user_id, device_id) | Even distribution across partitions |
| Table design | Single-table | Fewer round trips, lower cost, one schema |
| Read consistency | Eventually consistent by default | 2× cheaper; strong only where needed |
| GSI capacity | Equal to or higher than base WCU | Avoid base table throttling from GSI backpressure |
| Transactions | Use sparingly (2× cost) | Only for cross-item atomicity; use conditional writes otherwise |
| Global Tables | Active-active with regional ownership | Avoid conflict resolution complexity |
| Streams | NEW_AND_OLD_IMAGES for audit; KEYS_ONLY for cache invalidation | Minimize stream size |
| Hot partition fix | Write sharding (random suffix) or adaptive capacity | Distribute writes; DynamoDB handles reads adaptively |

---

## Interview Scenarios

### "Hot partition — one seller_id gets 80% of all reads (flash sale)"
- DynamoDB throttles partitions exceeding their WCU/RCU allocation
- Fix option A: DAX in front of DynamoDB — cache reads; hot partition reads served from DAX, not DynamoDB
- Fix option B: write sharding — store item as `seller_id#0` through `seller_id#9`; read all shards and merge
- Fix option C: GSI overloading — route reads to a GSI with different partition key that spreads the hot data
- Adaptive capacity handles moderate hot partitions; extreme hot keys require explicit sharding

### "Need to query by multiple access patterns (by email, by phone, by region)"
- DynamoDB requires knowing access patterns at design time; each GSI supports one access pattern
- For 3 access patterns: create 3 GSIs (email → PK, phone → PK, region → PK)
- GSI cost: same WCU/RCU as base table; 3 GSIs = ~4× the write cost for heavily written items
- Alternative: write to DynamoDB Streams → downstream sync to Elasticsearch for flexible querying

### "Transaction fails halfway through — item A updated, item B not updated"
- DynamoDB Transactions are all-or-nothing: if any condition fails, entire `TransactWriteItems` rolls back
- Use `ConditionExpression` on each item to detect conflicts before write
- Idempotency: include `ClientRequestToken` on `TransactWriteItems` → safe to retry without double-apply
- 25-item limit per transaction: for larger batches, use saga pattern (sequence of conditional writes + compensating actions)

### "Table grows to 10 TB — cost becomes prohibitive"
- Analyze access patterns: items accessed <1× per month are paying full Standard rate unnecessarily
- Enable DynamoDB Standard-IA (Infrequent Access) storage class → ~60% storage cost reduction
- TTL: set expiration on transient items (sessions, OTPs, temp tokens) → automatic free deletion
- Archive cold data: DynamoDB → S3 via DynamoDB export; query with Athena; delete from DynamoDB

### "Need to sync DynamoDB to Elasticsearch for full-text search"
- Enable DynamoDB Streams (NEW_AND_OLD_IMAGES)
- Lambda trigger on stream → writes to Elasticsearch / OpenSearch
- Lag: near-real-time (<1s typically); Lambda invoked per shard of stream
- On initial bootstrap: DynamoDB export to S3 → bulk import to Elasticsearch → then switch to stream

### "Constraint: query must return results sorted by timestamp"
- DynamoDB: items within a partition sorted by sort key
- Design: `PK = entity_type#entity_id`, `SK = timestamp#uuid` → range scan returns time-ordered items
- Reverse sort: scan with `ScanIndexForward=false` → newest first
- Cross-partition time sort impossible natively → use GSI with `PK = entity_type`, `SK = timestamp` — but this puts all items of a type in one partition (hot spot at scale)

### "Global Tables — users in EU write data but see US users' stale data"
- Global Tables use last-writer-wins (LWW) with DynamoDB timestamps; async replication lag ~1–2s
- If EU user reads immediately after US user writes: may see stale data during replication lag
- Fix: regional ownership — each user "owns" a home region; always read from home region; replicate for disaster recovery only
- For truly consistent global reads: direct all reads to one canonical region (defeats geo-distribution purpose)
