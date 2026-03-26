# System Design Interview Cheatsheet

## Interview Model
```
FR → NFR → Estimates → API → Data Model → HLD → Deep Dive → Trade-offs → Failure Modes
```
**Time split (45 min)**: FR+NFR (5) → Estimates (5) → API+Data Model (5) → HLD (10) → Deep Dive (15) → Trade-offs+Failures (5)

**Tool**: Excalidraw (or whiteboard)

---

## Step 1 — Clarifying Questions

Never assume. Ask before drawing anything.

| Category | Questions |
|----------|-----------|
| Scale | DAU? Peak QPS? Data volume? Growth rate? |
| Access pattern | Read-heavy or write-heavy? Read:write ratio? |
| Latency | p99 SLO? Real-time or batch acceptable? |
| Consistency | Strong or eventual? Read-your-writes needed? |
| Availability | 99.9% or 99.999%? Single-region or global? |
| Durability | Can we lose data? RPO/RTO requirements? |
| Data size | Avg object size? Retention period? |
| Scope | What to exclude? Mobile/web/both? |

---

## Step 2 — Functional Requirements (FR)

- Write **only core features** — not nice-to-haves
- Prioritize: P0 (must-have) vs P1 (if time allows)
- Example (URL shortener):
  - P0: shorten URL, redirect to original
  - P1: analytics, expiry, custom alias

---

## Step 3 — Non-Functional Requirements (NFR)

| NFR | Typical Target | Notes |
|-----|---------------|-------|
| Availability | 99.99% | Multi-region for 99.999% |
| Read latency | < 100ms p99 | Agree on specific SLO |
| Write latency | < 500ms p99 | Or async (202 Accepted) |
| Throughput | X writes/s, Y reads/s | Derived from estimates |
| Consistency | Eventual / Strong | Per feature |
| Durability | No data loss | Or define RPO |
| Scalability | Handle 10× growth | State the assumption |

---

## Step 4 — Capacity Estimation (80:20 Rule)

> 80% of traffic in 20% of the day → peak ≈ 2–3× average

```
DAU:             ___M
Avg QPS:         DAU × actions_per_day / 86,400 = ___k
Peak QPS:        avg × 3 = ___k

Write QPS:       ___k     Read QPS: ___k
Record size:     ___ KB

Daily storage:   write_QPS × 86,400 × record_size = ___ GB/day
5yr total:       daily × 1,825 × 3 (replicas) = ___ TB

Ingress BW:      write_QPS × request_size = ___ MB/s
Egress BW:       read_QPS × response_size = ___ GB/s
```

→ Full latency numbers + power-of-2 table: [capacity/capacity_planning.md](capacity/capacity_planning.md)

---

## Step 5 — API Design

**REST** for external-facing; **gRPC** for internal (binary, lower latency, streaming).
**GraphQL** when clients need flexible field selection (BFF pattern).

| Protocol | Use When |
|----------|----------|
| REST (HTTP/JSON) | Public API, browser clients, simple CRUD |
| gRPC | Internal microservices, streaming, mobile (protobuf smaller) |
| GraphQL | Flexible queries, mobile BFF, multiple client types |
| WebSocket | Bidirectional real-time (chat, live feed, gaming) |
| SSE | Server → client stream only (notifications, live dashboard) |

### Design Rules
- **Idempotency**: PUT/DELETE must be idempotent; POST needs `Idempotency-Key` header for retries
- **Pagination**: cursor-based (stable, scalable) over offset (breaks on inserts)
- **Versioning**: `/v1/` prefix; never break existing clients
- Return `202 Accepted` + job ID for async operations

```
# Example — URL shortener
POST /v1/urls
  Body: { long_url, ttl_seconds?, alias? }
  Response: { short_code, short_url, expires_at }

GET /{code}                           → 301/302 redirect
GET /v1/urls/{code}/analytics         → { clicks, countries[], referrers[] }
DELETE /v1/urls/{code}                → 204 No Content
```

---

## Step 6 — Data Model

### Which Database?

| System | Use When |
|--------|----------|
| **PostgreSQL** | ACID, complex queries, joins, geospatial (PostGIS), general-purpose |
| **MySQL** | Web apps, read replicas via binlog, wide ecosystem |
| **DynamoDB** | Managed, predictable latency, simple KV/document, autoscale |
| **Cassandra** | Multi-region writes, high write throughput, time-series, known access patterns |
| **MongoDB** | Flexible schema, nested documents, moderate consistency OK |
| **Redis** | Sub-ms reads, sessions, leaderboards, rate limiting, pub/sub, counters |
| **ClickHouse** | OLAP, columnar, analytical queries on billions of rows |
| **Elasticsearch** | Full-text search, log analytics, faceted search |
| **InfluxDB / TimescaleDB** | Time-series metrics, IoT, retention policies |
| **Neo4j** | Graph traversal, fraud detection, recommendations, social graph |
| **S3 / GCS** | Blob storage, large files, cheap durable storage, archival |

### Schema Design Rules
- Design around **access patterns**, not entities
- **Denormalize** for read performance — accept write complexity
- Partition key determines data distribution — avoid sequential keys (hot partition)
- Always include `created_at`, `updated_at`, soft-delete `deleted_at`
- Store large blobs in S3 — only URL in DB

---

## Step 7 — High-Level Design

```
Client → CDN → LB → API Gateway → Services → DB / Cache / Queue
                                           ↓
                                  Workers ← Message Queue
                                           ↓
                                  Object Store (S3)
```

### Component Checklist
- [ ] Client (web/mobile/API)
- [ ] CDN (static assets, large files, geo reads)
- [ ] Load balancer (L4 or L7)
- [ ] API Gateway (auth, rate limit, routing)
- [ ] App servers (stateless → horizontally scalable)
- [ ] Primary DB + read replicas
- [ ] Cache (Redis/Memcached)
- [ ] Message queue (Kafka/SQS) for async decoupling
- [ ] Background workers
- [ ] Object store (S3) for blobs
- [ ] Search index (Elasticsearch) if needed
- [ ] Monitoring + alerting (Prometheus, Grafana, PagerDuty)

---

## Step 8 — Deep Dive

Pick 2–3 hard components. Interviewer usually steers this.

| Common Deep Dives | Key Questions |
|-------------------|--------------|
| Write path | How does data flow from client → durable storage? |
| Read path | How is data fetched, cached, served efficiently? |
| Fan-out | How does one write reach N users? |
| Hotspot | What happens when one key/user gets 10× traffic? |
| Consistency | How do we handle replication lag? |
| Search | How is data indexed and queried at scale? |
| Unique ID gen | Snowflake, UUID, hash-based — which and why? |

---

## Patterns by Problem Type

### Read-Heavy (feed, catalog, search)
- Cache aggressively (Redis L1, CDN L2)
- Read replicas — route 80%+ reads away from primary
- Denormalize / precompute (materialized views)
- Serve stale if tolerable (TTL + async background refresh)
- Indexes on all query fields; covering indexes to avoid table scans

### Write-Heavy (logging, metrics, events, IoT)
- LSM-tree storage (Cassandra, RocksDB, InfluxDB) — sequential writes
- Message queue as write buffer (Kafka) → async batch flush to DB
- Partition by hash to spread load evenly
- Avoid sequential keys — use random prefix or hash to prevent hot partitions
- Return `202 Accepted` immediately; process async

### Real-Time / Low-Latency (chat, live feed, gaming)
- WebSockets for bidirectional persistent connections
- SSE for server → client push only
- Long polling as degraded fallback
- Pub/sub (Redis pub/sub, Kafka) for fan-out to connected clients
- In-memory state (Redis) for presence, typing indicators, online status

### Fan-Out at Scale (Twitter timeline, notifications)
| Model | Write Cost | Read Cost | Use When |
|-------|-----------|-----------|----------|
| Fan-out on write (push) | O(N followers) | O(1) | Read-heavy, followers < 10k |
| Fan-out on read (pull) | O(1) | O(following) | Write-heavy, celebrity accounts |
| **Hybrid** | Async push normal users | Pull for celebrities | Twitter/Instagram |

### Strong Consistency Required (payments, inventory, booking)
- Single-region SQL with transactions
- Serializable or SSI isolation
- Optimistic locking (`version` column + CAS) or `SELECT FOR UPDATE`
- Idempotency key on all write APIs (dedup window = 24h)
- Prefer saga pattern over 2PC for cross-service atomicity

---

## When to Use Which Tool

### Message Queue

| System | Use When | Avoid When |
|--------|----------|-----------|
| **Kafka** | High throughput, ordered events, replay, event sourcing, audit log | Need <10ms latency, simple task queue |
| **SQS** | Simple managed task queue, at-least-once OK, serverless | Strict ordering, replay |
| **SQS FIFO** | Strict ordering, exactly-once per group | High throughput (300 msg/s limit/group) |
| **Redis Streams** | Low latency, small scale, in-memory OK | Durability critical, large volumes |
| **RabbitMQ** | Complex routing (fanout/topic/direct), legacy | High throughput at scale |

### Cache

| System | Use When |
|--------|----------|
| **Redis** | Rich structures (sorted sets, hashes), pub/sub, Lua scripting, persistence option |
| **Memcached** | Simple KV only, max memory efficiency, multi-threaded, no persistence |
| **In-process (Caffeine)** | Sub-ms latency, JVM app, tolerate stale across instances |
| **CDN** | Static assets, large files, geo-distributed reads, staleness tolerated |

### Load Balancer

| Type | Layer | Knows | Use For |
|------|-------|-------|---------|
| L4 (NLB) | TCP/UDP | IP + port | Low latency, non-HTTP, raw throughput |
| L7 (ALB) | HTTP | URL, headers, cookies | HTTP routing, SSL termination, rate limiting, A/B |

### RPC / API Protocol

| Protocol | Latency | Payload | Streaming | Use |
|----------|---------|---------|-----------|-----|
| REST/HTTP | Medium | JSON (verbose) | No | Public APIs, browsers |
| gRPC | Low | Protobuf (compact) | Yes (bidirectional) | Internal services, mobile |
| GraphQL | Medium | JSON flexible | Subscriptions | BFF, flexible client queries |
| WebSocket | Very low | Binary/text | Bidirectional | Chat, gaming, live data |
| SSE | Low | Text stream | Server→client | Notifications, live feeds |

---

## Failure Modes & Resilience

### Circuit Breaker
```
CLOSED → [threshold failures] → OPEN (fast-fail)
   ↑                                    ↓
HALF-OPEN ←────── [cooldown expires] ───┘
  (probe one request → success → CLOSED, fail → OPEN)
```
Prevents cascade failure when downstream is slow/down. Used in: Resilience4j, AWS SDK.

### Rate Limiting Algorithms

| Algorithm | Behavior | Use |
|-----------|----------|-----|
| Token bucket | Allows bursts up to bucket capacity | API gateways, bursty traffic |
| Leaky bucket | Strict constant output rate | Traffic shaping |
| Fixed window | Simple counter per window | Easy but spike at window edge |
| Sliding window log | Exact, memory-heavy | High-precision |
| Sliding window counter | Approx, memory-efficient | Most production systems |

### Graceful Degradation
- Serve stale cache when DB is down
- Return partial results instead of full error
- Disable non-critical features (recommendations) when overloaded
- Queue writes instead of rejecting when downstream is slow

### SPOF Checklist
- [ ] DB: replica set or managed failover (RDS Multi-AZ)
- [ ] App servers: multiple instances behind LB
- [ ] Cache: Redis Sentinel or Redis Cluster
- [ ] Queue: Kafka cluster (3+ brokers) or managed SQS
- [ ] Region: multi-AZ minimum; multi-region for 99.999%
- [ ] External deps: timeout + fallback for every call

### Idempotency Pattern
```
Client → POST /payments { amount: 100, idempotency_key: "uuid-123" }
Server → check dedup store for "uuid-123"
       → if exists: return cached response
       → if new: process + store (key → response) with TTL
```
Used in: Stripe, AWS SDK, any payment or critical write API.

---

## Async vs Sync

| | Sync | Async |
|-|------|-------|
| Use when | Response needed immediately | Processing can be deferred |
| Client experience | Waits for result | Gets 202 + job ID immediately |
| Coupling | Tight | Loose |
| Failure handling | Caller handles instantly | Queue retries automatically |
| Examples | Auth, reads, payments | Email, notifications, analytics, media encoding |

---

## Pull vs Push

| | Push | Pull |
|-|------|------|
| Latency | Lower — server initiates | Higher — bounded by poll interval |
| Scale | Hard — server tracks N subscribers | Easy — consumers control their own rate |
| Miss risk | Yes, if consumer is down | No — consumer fetches when ready |
| Use | WebSocket, SSE, webhooks, notifications | Kafka consumer, cron jobs, RSS |

---

## CDN

- Use for: static assets (JS/CSS/images), video, large files, geo-distributed reads
- Cache-Control + Expires headers control edge TTL
- **Invalidation**: versioned filenames (`/app.v4.js`) preferred over purge API
- **Origin shield**: client → CDN edge → CDN POP → origin (reduces origin fan-in)
- **Stale-while-revalidate**: serve stale, refresh in background — no user-visible latency

---

## Common Trade-offs to Discuss

| Trade-off | Option A | Option B |
|-----------|----------|----------|
| Consistency vs availability | Strong (CP) — safer | Eventual (AP) — faster |
| Latency vs durability | Async write (fast) | Sync replicate (safe) |
| Normalization vs denormalization | Clean schema, slower reads | Redundant data, fast reads |
| Fan-out on write vs read | Fast reads, slow writes | Fast writes, slow reads |
| SQL vs NoSQL | Flexible queries, harder to shard | Fixed patterns, easy to shard |
| Monolith vs microservices | Simple ops, hard to scale parts | Complex ops, independent scale |
| Push vs pull delivery | Low latency | Consumer controls pace |
| Cache-aside vs write-through | Simple, possible cold start | Always warm, write overhead |
| Strong ID (UUID) vs Seq ID | No hotspot, no ordering | Ordered, hot partition risk |

---

## Hot Key / Skew Problems

| Problem | Symptom | Fix |
|---------|---------|-----|
| Hot partition | One shard at 100%, rest idle | Random key suffix, virtual shards |
| Celebrity / viral post | Fan-out to 100M followers spikes | Hybrid push/pull, async workers |
| Cache stampede | Hot key expires → DB hammered | Mutex on miss, probabilistic early expiry, background refresh |
| Write skew | Concurrent reads of shared resource → both write | `SELECT FOR UPDATE`, serializable isolation |
| Hot replica | All reads to one replica | Read replica pool + random routing |

---

## Red Flags Interviewers Watch For

- ❌ Single point of failure with no mitigation mentioned
- ❌ Storing large blobs (images/video) directly in DB
- ❌ Synchronous fan-out to millions of followers
- ❌ No caching for read-heavy workloads
- ❌ Sequential auto-increment keys on a sharded system
- ❌ Polling DB at high frequency instead of event-driven
- ❌ No idempotency on payment or critical write APIs
- ❌ Strong consistency everywhere (over-engineering where eventual is fine)
- ❌ Jumping to components without clarifying requirements first
- ❌ No mention of monitoring, logging, or alerting

## Green Flags

- ✅ Ask clarifying questions before drawing anything
- ✅ State assumptions explicitly ("I'll assume 100M DAU")
- ✅ Present trade-offs: "X vs Y — I chose X because..."
- ✅ Proactively mention what breaks at 10× and how to handle it
- ✅ Bring up hot keys, skew, and celebrity problem unprompted
- ✅ Design for failure: circuit breaker, graceful degradation, retry + idempotency
- ✅ Capacity numbers that justify architectural choices
- ✅ Know when NOT to distribute: "a single Postgres instance handles this fine at this scale"
