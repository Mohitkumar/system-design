# Amazon SQS — System Design

## What Is It
Fully managed distributed message queue. Decouples producers from consumers.
- **Standard Queue**: at-least-once delivery, best-effort ordering, nearly unlimited throughput
- **FIFO Queue**: exactly-once processing, strict ordering, 3,000 msg/s (with batching)

---

## Functional Requirements
- Send message (single + batch up to 10)
- Receive message (poll, long poll, batch)
- Delete message (explicit ack after processing)
- Change message visibility timeout (extend lease mid-processing)
- Dead Letter Queue (DLQ) after N failed processing attempts
- Delay queues (delay delivery up to 15 min)
- Message attributes (metadata key-value pairs)
- FIFO: deduplication by MessageDeduplicationId, grouping by MessageGroupId

## Non-Functional Requirements
- **Availability**: 99.9% (multi-AZ by default)
- **Durability**: messages stored across multiple AZs
- **Latency**: < 10ms send; < 20ms receive
- **Scale**: standard = unlimited TPS; FIFO = 3,000 msg/s per queue (with batching)
- **At-least-once** (Standard) or **exactly-once** (FIFO)
- Message size: up to 256 KB; larger via S3 pointer (Extended Client Library)

---

## Capacity Estimation

```
Messages/day:    1 billion
Avg msg size:    1 KB
Write QPS:       1B / 86,400 ≈ 11,600/s
Peak QPS:        11,600 × 3 = ~35k/s (80:20 rule)

Storage (in-flight + retained):
  Default retention: 4 days (max 14 days)
  In-flight msgs:    max 120,000 per Standard queue
  Storage/day:       1B × 1 KB = 1 TB/day
  4-day retention:   ~4 TB

Bandwidth:
  Ingress:  35k/s × 1 KB = 35 MB/s
  Egress:   35k/s × 1 KB = 35 MB/s (each msg read once)

IOPS:
  Write: 35k msg/s × 1 sequential write = 35k sequential IOPS
         × 3 AZ replicas = 105k sequential writes/s across cluster
  Read:  35k receive/s; visibility lock update = 35k random writes on index
         + 35k delete/s = 70k random IOPS on in-flight tracker
  Sequential write (NVMe 1GB/s): 35 MB/s << 1 GB/s → write-throughput fine on 1 node
  Random IOPS (NVMe 500k): 70k IOPS << 500k → fine on 1 node
  With 3× replication: 3 storage nodes per host set

Server count:
  Frontend servers:  35k req/s ÷ 10k per server = 4 (run 10–15 for HA + AZ spread)
  Storage nodes:     3 per host set × N host sets (AWS runs many host sets per queue at scale)
  In-flight tracker: Redis cluster, 3 nodes (visibility lease store)
  Metadata service:  3–5 nodes (queue config, rarely changes)
```

---

## Architecture

```
Producer(s)                                    Consumer(s)
   │                                               │
   │ SendMessage                    ReceiveMessage │
   ▼                                               ▼
┌──────────────────────────────────────────────────────────┐
│                     SQS Service                          │
│                                                          │
│  ┌─────────────┐    ┌──────────────┐   ┌─────────────┐  │
│  │  Front-End  │    │  Metadata    │   │  Back-End   │  │
│  │  Servers    │───►│  Service     │───►  Storage    │  │
│  │  (stateless)│    │  (queue cfg, │   │  (message   │  │
│  └─────────────┘    │  dedup idx)  │   │   data)     │  │
│                     └──────────────┘   └─────────────┘  │
│                                              │            │
│                          ┌───────────────────┤            │
│                        AZ-1    AZ-2    AZ-3  │            │
│                        (replicated storage)  │            │
│                                              │            │
│  ┌────────────────────────────────────────┐  │            │
│  │  Visibility Timeout Manager            │  │            │
│  │  DLQ Redriver | Delay Scheduler        │  │            │
│  └────────────────────────────────────────┘  │            │
└──────────────────────────────────────────────────────────┘
                              │
                         Dead Letter Queue (DLQ)
                         (after maxReceiveCount failures)
```

---

## Core Concepts

### Message Lifecycle

```
Producer                    SQS                         Consumer
   │                         │                              │
   │── SendMessage ──────────►│                              │
   │                         │ store + replicate (3 AZs)    │
   │                         │                              │
   │                         │◄─── ReceiveMessage ──────────│
   │                         │                              │
   │                         │─── return msg + receiptHandle►│
   │                         │                              │
   │                         │  msg INVISIBLE (visibility   │
   │                         │  timeout = 30s default)      │
   │                         │                              │
   │                         │◄─── DeleteMessage(receipt) ──│ (success)
   │                         │  msg permanently deleted     │
   │                         │                              │
   │         OR              │                              │
   │                         │  timeout expires (no Delete) │
   │                         │  msg becomes VISIBLE again ──►│ (retry)
```

### Visibility Timeout
- Window during which a received message is **invisible** to other consumers
- Consumer must `DeleteMessage` before timeout → prevents duplicate processing
- Default: 30s; Max: 12 hours
- Extend mid-processing: `ChangeMessageVisibility` (for long jobs)
- **Idempotent consumers required** — Standard queues can deliver same message twice

### Long Polling
```
Short poll (default): returns immediately even if queue empty → wasted API calls
Long poll:            waits up to 20s for a message to arrive → fewer empty responses
                      WaitTimeSeconds=20 on ReceiveMessage call
```
Always use long polling in production — reduces cost and CPU.

---

## Standard Queue Deep Dive

### At-Least-Once Delivery
- Messages stored across multiple servers/AZs
- On receive: one storage node returns message, others don't know yet
- If that node fails before Delete → other nodes re-deliver → **duplicate**
- **Consumers must be idempotent** (use message ID as dedup key in your DB)

### Best-Effort Ordering
- SQS does not guarantee order in Standard queues
- Messages may arrive out of order due to distributed storage
- If ordering matters → use FIFO queue or include sequence number in message

### Throughput
- Effectively unlimited — SQS shards queues internally
- AWS adds more partitions as traffic increases (transparent)
- No pre-provisioning needed

---

## FIFO Queue Deep Dive

### Exactly-Once + Ordered
```
MessageGroupId:          logical group — messages within same group ordered strictly
                         different groups processed independently and in parallel

MessageDeduplicationId:  dedup window = 5 minutes
                         same dedup ID within 5 min → second send silently discarded
                         (SHA-256 of body used if ContentBasedDeduplication=true)
```

### Throughput Limits
| Mode | TPS |
|------|-----|
| Without batching | 300 msg/s per queue |
| With batching (10 msg/batch) | 3,000 msg/s per queue |
| High-throughput FIFO (2023) | 70,000 msg/s per API action |

### Ordering Guarantee
```
Group "user-123":   msg1 → msg2 → msg3  (strict order within group)
Group "user-456":   msgA → msgB         (strict order within group)

Groups processed concurrently — scale by adding more MessageGroupIds
```

---

## Dead Letter Queue (DLQ)

```
Source queue config:
  maxReceiveCount: 3   ← message moves to DLQ after 3 failed deliveries

Flow:
  Message received → processing fails → visibility timeout expires → re-queued
  After 3rd failure → moved to DLQ automatically

DLQ:
  Same type as source (Standard → Standard DLQ, FIFO → FIFO DLQ)
  Retention: up to 14 days for investigation
  Alarm: CloudWatch metric on DLQ depth → PagerDuty alert
  Redrive: replay DLQ messages back to source queue after bug fix
```

---

## Delay Queue / Message Timers

```
Delay queue:     all messages delayed by X seconds (0–900s / 15 min)
                 Use: schedule jobs, debounce, retry backoff

Message timer:   per-message delay (overrides queue delay)
                 SendMessage(..., DelaySeconds=60)
                 Use: "process this in 1 minute"
```

---

## Large Messages (>256 KB)

SQS max message size = 256 KB.

**S3 Extended Client Library pattern:**
```
Producer:
  1. PUT large payload → S3 (returns S3 key)
  2. SendMessage → SQS with S3 pointer: { "s3Bucket": "x", "s3Key": "y" }

Consumer:
  1. ReceiveMessage → get S3 pointer
  2. GET object from S3
  3. Process full payload
  4. DeleteMessage from SQS + optionally delete from S3
```

---

## SQS vs Kafka vs RabbitMQ

| | SQS Standard | SQS FIFO | Kafka | RabbitMQ |
|-|-------------|----------|-------|----------|
| Ordering | Best-effort | Strict per group | Strict per partition | Per queue |
| Delivery | At-least-once | Exactly-once | At-least-once (configurable) | At-least-once |
| Throughput | Unlimited | 3k–70k/s | Millions/s | Tens of thousands/s |
| Retention | 1min–14 days | 1min–14 days | Configurable (forever) | Until consumed |
| Replay | No | No | Yes (consumer group offset) | No |
| Pull/Push | Pull (polling) | Pull | Pull | Push + Pull |
| Routing | None | None | Topic/partition | Exchange (topic/fanout/direct) |
| Ops | Fully managed | Fully managed | Self-managed or MSK | Self-managed |
| Use when | Simple task queue, async decoupling | Ordered processing, dedup | Event streaming, replay, audit | Complex routing, RPC patterns |

---

## SQS Internal Architecture (Inferred from AWS Papers)

### Storage Layer
- Messages stored in a **distributed, replicated storage system** (custom, not off-the-shelf)
- Each queue sharded across multiple **host sets** (groups of servers)
- Each host set: primary + replicas across 3 AZs
- Write: majority ACK before confirming send to producer

### Metadata Layer
- Queue configuration (visibility timeout, delay, DLQ config, policy) in metadata service
- FIFO deduplication index: hash(MessageDeduplicationId) → expiry timestamp
- Visibility lease table: messageId → {receiptHandle, expiry, receiveCount}

### Receive Path
```
ReceiveMessage(QueueUrl, MaxNumberOfMessages=10, WaitTimeSeconds=20):
  1. Front-end picks a host set for this queue shard
  2. Query storage: find VISIBLE messages (not locked, not delayed)
  3. Lock messages: set visibility timeout lease
  4. Return messages + receiptHandles (signed tokens encoding message ID + expiry)
  5. Background: if WaitTimeSeconds > 0, hold connection and poll every ~1s
```

### ReceiptHandle
```
receiptHandle = sign({queue_id, message_id, shard_id, receive_timestamp, expiry}, secret)
→ opaque token to client; server validates on Delete/ChangeVisibility
→ different each time same message is received (prevents replay of old handles)
```

---

## Key Design Patterns

### Fan-Out: SNS → Multiple SQS Queues
```
Publisher → SNS Topic → [SQS Queue A (email service),
                          SQS Queue B (analytics service),
                          SQS Queue C (notification service)]
```
Each subscriber gets its own copy. SNS delivery to SQS is at-least-once.

### Work Queue (Task Distribution)
```
N producers → SQS → M workers (auto-scaled based on queue depth)
CloudWatch metric: ApproximateNumberOfMessagesVisible
→ Auto Scaling target tracking: keep metric at X msgs per instance
```

### Request-Response (RPC over SQS)
```
Client:   create temp reply queue (or use SQS temporary queue)
          send request with replyTo=queue_url, correlationId=uuid
Server:   process request, send response to replyTo queue
Client:   poll replyTo queue, match correlationId
```
Rarely used — HTTP/gRPC better for synchronous RPC. SQS RPC: latency > 100ms.

---

## Failure Handling

| Failure | SQS Behavior |
|---------|-------------|
| Consumer crashes | Visibility timeout expires → message re-delivered |
| Consumer slow | `ChangeMessageVisibility` to extend lease |
| Repeated failures | maxReceiveCount → moves to DLQ |
| SQS node failure | Replicated across AZs → transparent failover |
| Region failure | SQS in one region — use multi-region if needed |
| Duplicate messages | Consumers must be idempotent |

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Pull vs push | Pull (consumer polls) | Consumer controls rate; backpressure naturally |
| At-least-once | Default Standard | Simpler, higher throughput; consumers handle dedup |
| Visibility timeout | Locks message during processing | Prevents parallel processing without distributed lock |
| DLQ | Separate queue after N failures | Isolate poison pills; don't block healthy messages |
| Long polling | WaitTimeSeconds=20 | Reduce empty receives; lower cost; lower latency |
| No replay | Messages deleted on ACK | Simple; use Kafka if replay needed |

---

## Interview Scenarios

### "Messages being processed multiple times — consumers see duplicates"
- Standard SQS: at-least-once delivery; duplicates expected during redrive and AZ failover
- Fix: consumer must be idempotent — use `MessageId` as idempotency key on DB upsert
- Or: switch to FIFO queue — exactly-once within 5-minute dedup window using `MessageDeduplicationId`
- Monitor `NumberOfMessagesSent` vs downstream DB write count — divergence reveals duplicate rate

### "Consumer processes a message but takes 45 minutes — visibility timeout expires, message redelivered"
- Default visibility timeout: 30s — too short for long jobs
- Fix option A: set visibility timeout at receive time: `ReceiveMessage(VisibilityTimeout=3600)`
- Fix option B: consumer heartbeat — call `ChangeMessageVisibility` every 5 minutes to extend lease
- Option B is safer: if consumer crashes, heartbeat stops → timeout expires → redelivery happens correctly

### "Poison pill — one message always fails, blocks queue processing"
- Consumer retries same message; hits `maxReceiveCount` (e.g., 5) → SQS moves it to DLQ automatically
- DLQ alarm: CloudWatch alarm on `ApproximateNumberOfMessagesVisible > 0` → notify on-call
- Inspect DLQ: parse failed message, fix bug in consumer code, redrive from DLQ back to source queue
- Never delete DLQ messages without processing or investigation — they represent real work items

### "Need strict FIFO ordering within a logical group (e.g., per-user event ordering)"
- FIFO queue + `MessageGroupId = user_id` — SQS guarantees FIFO within each group
- Different groups processed in parallel; same group processed sequentially by one consumer
- Throughput limit: FIFO = 3,000 msg/s with batching (vs Standard = unlimited)
- Trade-off: if one group has a slow consumer, it blocks only that group (no head-of-line for other groups)

### "Traffic spikes 100× for 5 minutes (flash sale) — consumers overwhelmed"
- SQS acts as buffer — messages pile up in queue rather than hitting backend directly
- Consumer auto-scaling: Lambda or EC2 ASG scaled by `ApproximateNumberOfMessagesVisible` metric
- Lambda + SQS: Lambda scales out consumers automatically up to concurrency limit
- Set `maxBatchSize` on consumer: process 10 messages per invocation → reduce consumer overhead

### "Constraint: messages must be processed in a specific order that depends on content (not just group)"
- SQS FIFO + `MessageGroupId` provides ordering by group, not content-defined order
- For content-dependent ordering: consumer must implement sequencing logic — include `sequenceNumber` in message; consumer buffers out-of-order messages, processes in sequence
- Alternative: use Kafka (strict per-partition order) if content-defined ordering is central to the system

### "Need to schedule a task 30 days in the future"
- SQS `DelaySeconds` max = 15 minutes — too short
- Pattern: write task to DynamoDB with `executeAt` timestamp; EventBridge Scheduler triggers Lambda at time T
- Or: Step Functions `Wait` state — workflow pauses until specified time, then resumes
- SQS-only workaround: re-enqueue message with delay each time it's received, until `executeAt` is reached (polling loop — expensive, not recommended)
