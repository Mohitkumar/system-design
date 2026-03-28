# Distributed Logging System

**Examples:** AWS CloudWatch Logs, Datadog, Splunk, Elastic Stack (ELK), Grafana Loki

---

## Concepts

### Log Pipeline Stages
```
Emitter (app) → Collector → Buffer (Kafka) → Indexer → Query Layer
                                    │
                                    └──► Cold Store (S3)
```

### Structured vs Unstructured Logs
| | Unstructured | Structured |
|--|---|---|
| Format | Free text: `2024-01-01 ERROR db timeout` | JSON: `{"ts":1704067200,"level":"ERROR","msg":"db timeout","service":"api"}` |
| Search | Full-text only | Field-level filter + full-text |
| Ingestion cost | Low (pass-through) | Higher (parse + index fields) |
| Query power | grep-style | SQL-like + aggregations |

### Hot vs Cold Log Storage
```
< 7 days  → Hot tier  (indexed, fast query, SSD-backed)  ← recent debugging
7–30 days → Warm tier (indexed, slower query, HDD)       ← incident retro
> 30 days → Cold tier (S3 Glacier, gzip, no index)       ← compliance archive
```

---

## Functional Requirements

- Ingest logs from any source: applications, containers, OS agents
- `putLogs(logGroupName, logStreamName, logEvents[])` — batch write
- `filterLogs(logGroup, filter, startTime, endTime)` → paginated log events
- Full-text search + field filters across log groups
- Log insights: aggregation queries (`count`, `avg`, `percentile` over fields)
- Metric extraction: derive CloudWatch metrics from log patterns
- Alerts: trigger on log pattern match or threshold (error rate > N/min)
- Log retention policy per log group (7 days / 30 days / 1 year / forever)

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Ingest latency | Log visible within 5s of emission |
| Query latency | < 5s for last-hour queries; < 30s for 30-day range |
| Availability | 99.99% for ingestion; 99.9% for query |
| Durability | No log loss after `putLogs` returns 200 |
| Scale | 10 PB/day ingest, 1M log producers |
| Consistency | Eventual — log ordering within stream; best-effort across streams |

---

## Capacity Estimation

**Assumptions:** 1M services, avg 1 KB/log line, 100K log lines/sec avg, peak 3× = 300K lines/sec.

**Ingest throughput:**
```
80:20 rule: peak = (total × 0.8) / (0.2 × 86400)
avg lines/sec = 1M services × 100 lines/min / 60 = ~1.6M lines/sec avg
peak = 1.6M × 2.5 = 4M lines/sec
Data rate: 4M × 1 KB = 4 GB/sec ingestion peak
```

**Storage:**
```
Daily raw logs:    4M lines/sec × 86400s × 1 KB × 0.4 (gzip) = ~138 TB/day compressed
Hot tier (7 days): 138 TB × 7 = ~1 PB  (SSD-backed index nodes)
Warm tier (30d):   138 TB × 30 = ~4 PB (HDD)
Cold tier (1yr):   138 TB × 365 = ~50 PB (S3 Glacier)
Index overhead:    ~30% of raw → hot tier index = 300 TB additional
```

**IOPS:**
```
Ingest write IOPS:
  4M lines/sec → batch into 1s segments → 4M writes/s in-memory
  Segment flush: 4 GB/sec sequential → NVMe (1 GB/s) → 4 ingest nodes minimum
  With replication (3×): 12 ingest nodes

Query read IOPS:
  10K concurrent queries × 10% cache miss × 3 disk reads/miss = 30K random IOPS
  SSD (100K IOPS/node) → 1 node handles; run 5 for headroom
```

**Server count:**

| Role | Count | Sizing |
|---|---|---|
| Log collectors (agents) | 1 per host (sidecar) | 2 cores, 512 MB RAM |
| Collector aggregators | 50 | 16 cores, 32 GB RAM, 10 Gbps NIC |
| Kafka brokers (buffer) | 20 | 16 cores, 64 GB RAM, 2× 2TB NVMe |
| Indexer nodes | 20 | 16 cores, 64 GB RAM, 2× NVMe |
| Hot query nodes | 60 (20 shards × 3 replicas) | 16 cores, 128 GB RAM, 2× 2TB NVMe SSD |
| Cold store | S3 | Managed |
| Cluster manager | 3 | 8 cores, 32 GB RAM |

---

## High-Level Architecture

```
App/Container/OS
      │
  [Log Agent]  (Fluentd / CloudWatch Agent / Filebeat)
      │  batch + compress
      ▼
[Collector Aggregator]  ── validates, enriches, routes by log group
      │
      ▼
[Kafka]  (durable buffer, decouples ingestion from indexing)
   │         │
   │         └──► [S3 writer]  ── raw gzip → cold store (S3)
   ▼
[Indexer]  (parses JSON fields, tokenizes text, builds inverted index)
      │
      ▼
[Index Shards]  (document-partitioned, 1 primary + 2 replicas)
   Shard 0   Shard 1  ...  Shard N
      │
      ▼
[Query Router]
      │
  ┌───┴───────────────┐
  ▼                   ▼
[Search API]    [Insights API]  (aggregation queries)
      │                │
      └───────┬─────────┘
              ▼
       [Alert Engine]
```

---

## Component Deep Dive

### 1. Log Agent (Collector)

Runs as a sidecar or daemonset on every host.

```
Responsibilities:
  - Tail log files / capture stdout/stderr from containers
  - Buffer locally (ring buffer, 64 MB) → handles brief collector outage
  - Batch: collect 1s worth of logs or 1 MB, whichever comes first
  - Compress: gzip batches (4–6× compression on text logs)
  - Retry with exponential backoff on collector failure
  - Tag with: service name, host, region, container ID, log group

Local buffer prevents log loss during:
  - Collector aggregator restart
  - Network blip < 60s
```

### 2. Collector Aggregator

Stateless fan-in layer. Receives batches from thousands of agents.

```
On receive:
  1. Decompress + parse log group / stream from metadata
  2. Validate: reject malformed batches; enforce max log line size (256 KB)
  3. Enforce rate limit per log group (prevent noisy tenant from starving others)
  4. Route to Kafka partition: hash(log_group_name) → consistent partition
  5. Produce to Kafka → ack to agent

Kafka partition key = log_group_name ensures:
  - All logs from same service → same Kafka partition → in-order delivery
  - Indexer consumer reads in order → stream ordering preserved
```

### 3. Kafka Buffer

Decouples ingestion rate from indexing rate. Acts as a durable, replayable buffer.

```
Topic: logs-ingest
  Partitions: 200  (handle 4 GB/sec ÷ 20 MB/sec per partition)
  Replication: 3
  Retention: 48 hours (replay window for indexer catch-up)
  Compression: LZ4 (low CPU, good ratio)

S3 sink consumer (parallel):
  Reads from Kafka → batches by log_group + hour → writes to S3
  S3 path: s3://logs-cold/{account}/{log_group}/{year}/{month}/{day}/{hour}.gz
  This runs independently of indexer — cold archive is always complete

Indexer consumer:
  Reads from Kafka → parses + indexes → writes to index shards
  Consumer group offset tracked in Kafka → replayable on indexer failure
```

### 4. Indexer

Transforms raw log events into queryable inverted index entries.

```
Per log event:
  1. Parse timestamp (nanosecond precision, store as int64)
  2. Parse JSON fields if structured → index each field separately
     {"level":"ERROR","service":"api","latency_ms":342}
     → field:level=ERROR, field:service=api, field:latency_ms=342
  3. Tokenize message field → inverted index terms
  4. Partition: hash(log_group_name + stream_name) % num_shards → target shard
  5. Write: (term, docID, position) to shard's in-memory segment

Segment lifecycle (same as Lucene):
  In-memory segment (write buffer, <1s) → immutable on-disk segment → merged segments

Near-real-time: flush every 1s → logs visible within ~2s of Kafka consumption
```

### 5. Index Shards (Hot Query Layer)

**Document partitioned** — same model as distributed search.

Each document = one log event with fields:
```
{
  _id:        UUID (log event ID)
  _ts:        int64 (nanoseconds epoch)
  log_group:  string (primary filter — all queries scoped to a log group)
  stream:     string
  message:    string (full-text indexed)
  level:      keyword (not tokenized)
  service:    keyword
  host:       keyword
  fields:     map<string, string>  (parsed JSON fields)
}
```

**Time-based segment pruning**: queries always include time range → index skips segments outside range (like Parquet column stats). Avoids full scatter for recent queries.

```
Query: log_group="payments" AND level="ERROR" AND @timestamp > now-1h

Router:
  1. Filter: all shards contain data for this log_group (document partitioned)
  2. BUT: time range pushdown → each shard skips segments older than 1h
  3. Scatter: 20 shards × top-1000 results each
  4. Merge: global top-1000 by timestamp desc
  5. Return paginated results
```

### 6. Cold Storage Query (S3)

For queries spanning > 7 days (warm/cold tier):

```
S3 path: s3://logs-cold/{account}/{log_group}/{year}/{month}/{day}/{hour}.gz

Query execution:
  1. Router determines time range falls in cold tier
  2. List S3 objects matching path prefix + time range
  3. Spin up ephemeral query workers (Lambda or spot EC2)
  4. Each worker: download + decompress + grep/parse matching objects
  5. Merge + paginate results back to user

Performance: 30-day query may scan 720 hourly files → parallelize across 50 workers
Latency: 10–30s (acceptable for historical analysis)
Cost optimization: S3 Select pushes filter to S3 → reduces data transferred
```

### 7. Insights / Aggregation Queries

SQL-like aggregations over log fields (CloudWatch Logs Insights equivalent).

```sql
fields @timestamp, service, latency_ms
| filter level = "ERROR"
| stats avg(latency_ms), count() by service
| sort count() desc
| limit 20
```

Execution:
```
Parse query → build query plan (filter + aggregate + sort + limit)
Scatter to all shards: each returns partial aggregates (count per service, sum of latency)
Merge: combine partial aggregates → final result
Time range: pushes to segment pruning → only scan relevant time window
```

### 8. Alert Engine

```
Alert definition:
  log_group: "payments"
  filter: level="ERROR"
  condition: count() > 100 per 5 minutes
  action: SNS → PagerDuty

Execution (streaming):
  Kafka consumer reads ingested logs (real-time, not from index)
  Sliding window counter per alert rule (Redis sorted set)
  On threshold breach → fire alert → deduplicate (suppress repeat for 15 min)

Why Kafka not index: alerts need <10s latency; index has ~2s lag but querying adds more
```

---

## Replication & Durability

```
Write path durability:
  1. Agent acks only after Kafka producer acks (Kafka replication factor=3)
  2. S3 writer runs in parallel → cold archive is always complete regardless of indexer health
  3. Indexer can replay from Kafka (48h retention) if it falls behind or crashes

Index replication:
  Each shard: 1 primary + 2 replicas (same as distributed search)
  Semi-sync: primary acks after 1 replica confirms
  Read from any replica → 3× read throughput
```

---

## Multi-Tenancy & Isolation

```
Log Group = tenant isolation boundary

Per log group:
  - Ingest rate limit (e.g., 5 MB/sec per group) → enforced at collector aggregator
  - Storage quota → retention policy enforced by TTL on index documents + S3 lifecycle
  - Query rate limit → query router enforces per-account QPS

Kafka partitioning by log_group:
  - One noisy log group fills its own Kafka partition buffer
  - Does not affect other log groups' partitions
```

---

## Failure Modes

| Failure | Impact | Mitigation |
|---|---|---|
| Agent crash | Logs lost since last flush (<1s buffer) | Local ring buffer; restart recovers in-flight batch |
| Collector aggregator down | Agent buffers locally (64 MB, ~60s) | Stateless; load balancer routes to healthy aggregator |
| Kafka broker down | Partition leader election (<10s) | 3 replicas; producer retries with backoff |
| Indexer crash | Log lag grows; no new logs visible | Stateless; restart + replay from Kafka offset; 48h replay window |
| Index shard primary down | Writes stall briefly; reads from replicas | Replica promotion <30s; reads unaffected |
| S3 unavailable | Cold archive writes queue in Kafka | Kafka 48h buffer absorbs S3 outage; retry on recovery |
| Query node OOM | Shard temporarily unavailable | Hedged requests; other replicas serve; restart clears memory |

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Kafka buffer | Between collector and indexer | Decouples ingest rate from index rate; replay on indexer failure; parallel S3 sink |
| S3 as cold store | Parallel to indexing, not dependent on it | Archive always complete; independent of index health |
| Document partitioning | Scatter/gather on query | Avoids hot-term shards; write path is single-shard per doc |
| Time-range pruning | Segment min/max timestamp metadata | Avoid full scatter; recent queries only touch recent segments |
| Per log-group Kafka partition | hash(log_group) | Ordering within stream; noisy tenant isolation |
| Streaming alerts | Kafka consumer, not index | <10s latency without index query overhead |
| Near-real-time (NRT) | 1s segment flush | Logs visible within 2–3s; balance vs merge overhead |

---

## Interview Scenarios

### "Logs are delayed — new errors take 5 minutes to appear in search"
- Pipeline bottleneck: collector → Kafka → indexer → shard flush
- Instrument each stage: measure Kafka consumer lag (`kafka_consumer_lag` metric)
- Common cause: indexer falling behind → add indexer nodes (stateless, just add consumers)
- Flush interval too high: reduce from 1s to 100ms for hot log groups
- Quick fix: query Kafka directly (bypass index) for last-5-minutes window

### "One service floods the pipeline — 100× normal log volume (noisy tenant)"
- Collector aggregator enforces per-log-group rate limit → excess dropped or sampled
- Kafka partition for that log group fills → backpressure to collector → agent buffers
- Does NOT affect other log groups on different Kafka partitions
- Alert: log_group ingest rate > threshold → notify service owner; auto-throttle

### "30-day search query times out (cold tier)"
- S3 query: 720 hourly files × decompress + scan = slow
- Fix: partition S3 objects by field value (service, level) if query pattern known
- Use S3 Select: push filter into S3 → 80% less data transferred
- Pre-aggregate: for common range queries, maintain rollup summaries (hourly counts per level per service) in a separate table
- Increase parallelism: 50 → 200 Lambda workers for cold scan

### "Need to search across log groups (global search)"
- Default: queries scoped to one log group (partition key in shard)
- Cross-group: scatter to ALL shards without log_group filter → expensive
- Fix: add a global shard tier (smaller, sampled) for cross-group search
- Or: maintain a separate cross-group index with just `level=ERROR` events

### "Traffic grows 10× — 40 GB/sec ingest"
- Kafka: scale partitions from 200 → 2000; add brokers
- Collector aggregators: stateless → add nodes; scale behind LB
- Indexers: add consumers (Kafka consumer group auto-rebalances)
- Index shards: pre-shard aggressively (200 shards vs 20); adding shards requires reindex
- S3: scales transparently; add prefix randomization if PUT rate hits S3 limits

### "Constraint: logs must be retained for 7 years (compliance)"
- Hot/warm tier: too expensive for 7 years (PB-scale on SSD/HDD)
- Cold tier: S3 Glacier Deep Archive at ~$0.00099/GB/month → 50 PB = ~$50K/month
- S3 Glacier Vault Lock: WORM (write once read many) for compliance — objects immutable after lock
- Access pattern: compliance queries rare → accept 12-hour Glacier retrieval latency
- Index: no indexing for compliance tier; grep on restore only

### "Alert fires but on-call can't query logs — index is down"
- Alert engine reads from Kafka (independent of index) → alerts still fire
- Emergency log access: query S3 cold store directly (always complete, always available)
- Provide runbook: `aws s3 cp s3://logs/{group}/{date}/*.gz . && zcat *.gz | grep ERROR`
- Design principle: S3 cold store is the source of truth; index is a performance layer
