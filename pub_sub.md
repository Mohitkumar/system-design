# Pub/Sub System

**Examples:** Apache Kafka, Google Pub/Sub, AWS SNS+SQS, Redis Pub/Sub, NATS

---

## Concepts & Theory

### Pub/Sub vs Message Queue

| | Message Queue (SQS) | Pub/Sub (Kafka) |
|---|---|---|
| Consumers | One consumer per message | Many consumer groups, each gets every message |
| Retention | Deleted after ack | Retained for configurable window (days/weeks) |
| Replay | No | Yes — consumers can rewind offset |
| Coupling | Producer knows queue name | Producer knows topic; consumers subscribe independently |
| Ordering | Per-queue FIFO | Per-partition ordered |
| Use case | Task distribution, job queues | Event streaming, fan-out, audit log, ETL |

### Core Model

```
Producer → [ Topic ] → [ Partition 0 ] → Consumer Group A (subscriber 1)
                   → [ Partition 1 ] → Consumer Group A (subscriber 2)
                   → [ Partition 2 ] → Consumer Group B (subscriber 1)
                                     → Consumer Group C (subscriber 1)
```

- **Topic** — logical channel; producers write to it, consumers subscribe to it.
- **Partition** — ordered, append-only log. Unit of parallelism and replication.
- **Offset** — position of a message within a partition. Consumers track their own offset.
- **Consumer Group** — a set of consumers that together consume a topic. Each partition assigned to exactly one member in the group. Different groups are independent.

### Message Retention & Replay

Messages are **not deleted on consume**. They expire after `retentionMs` (e.g. 7 days) or when storage limit is hit. Any consumer group can rewind to any offset — enables:
- Reprocessing after a bug fix
- New services bootstrapping from historical data
- Audit and compliance logs

### Push vs Pull Delivery

| | Push | Pull |
|---|---|---|
| How | Broker pushes to subscriber endpoint | Consumer polls broker for new messages |
| Backpressure | Hard — broker overwhelms slow consumer | Natural — consumer controls its rate |
| Latency | Lower (immediate push) | Slightly higher (poll interval) |
| Used by | Google Pub/Sub (push subscription), webhooks | Kafka, SQS |

Kafka uses **pull** — consumers call `fetch(topic, partition, offset, maxBytes)`.

### Delivery Guarantees

| Guarantee | Meaning | How |
|---|---|---|
| At-most-once | May lose, never duplicates | Commit offset before processing |
| At-least-once | May duplicate, never loses | Commit offset after processing — **standard** |
| Exactly-once | No loss, no duplicates | Idempotent producer + transactional consumer (Kafka EOS) |

### Fan-out Pattern

One event → multiple independent consumers:

```
Order Placed event
    ├─ Consumer Group: Inventory Service  (reduce stock)
    ├─ Consumer Group: Notification Service (send email)
    ├─ Consumer Group: Analytics Service  (update dashboard)
    └─ Consumer Group: Fraud Service      (score the order)
```

Each group maintains its own offset; one slow group doesn't affect others.

---

# Pub/Sub — System Design (Kafka-like)

---

## Functional Requirements

- `createTopic(name, partitions, replicationFactor)`
- `publish(topic, key?, message)` → offset
- `subscribe(topic, consumerGroup)` → stream of messages
- `commitOffset(topic, partition, consumerGroup, offset)` — ack processed messages
- `seek(topic, partition, consumerGroup, offset)` — replay from arbitrary offset
- At-least-once delivery; support for exactly-once via idempotent producers

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Throughput | 1M msg/sec write, 5M msg/sec read (fan-out) |
| Latency | Publish < 5ms p99, end-to-end < 20ms p99 |
| Durability | No data loss after publish returns |
| Availability | 99.99% |
| Retention | Up to 30 days |
| Scalability | Linear scale-out by adding partitions/brokers |

---

## Capacity Estimation

**Assumptions:** 500K producers, 10 consumer groups, 100M events/day, avg 1 KB/event.

- **80/20 rule:** peak = (100M × 0.8) / (0.2 × 86,400) ≈ **~4,600 msg/sec** average peak; design for 10× headroom = **~50K msg/sec**
- **Fan-out read load:** 50K msg/sec × 10 consumer groups = **500K reads/sec**
- **Storage:** 100M × 1 KB = 100 GB/day × 7 day retention = **700 GB**; 3× replication = **2.1 TB**
- **Bandwidth:** 50K × 1 KB = **50 MB/sec** inbound; 500 MB/sec outbound

**Brokers needed:** Each broker handles ~100 MB/sec disk I/O → **5 read brokers** minimum; with replication overhead, deploy **~10–15 brokers**.

---

## High-Level Architecture

```
Producers
    │  publish (TCP / gRPC)
    ▼
[ Broker Cluster ]
  ┌─────────────────────────────────────────┐
  │  Broker 0    Broker 1    Broker 2  ...  │
  │  [Topic A, P0 Leader]                   │
  │  [Topic A, P1 Follower]                 │
  │  [Topic B, P0 Follower]                 │
  └─────────────────────────────────────────┘
         │              │
         ▼              ▼
  [ ZooKeeper / KRaft ]     [ Offset Store ]
  cluster metadata,          consumer group offsets
  leader election            (stored in __consumer_offsets topic)

Consumers (pull)
    │ fetch(topic, partition, offset)
    ▼
[ Broker — Partition Leader ]
    └─ returns batch of messages
```

---

## Low-Level Design

### Partition Storage (Commit Log)

Each partition is stored as a **segmented append-only log** on the broker's disk:

```
/data/topic-A/partition-0/
  00000000000000000000.log      ← messages, segment 1
  00000000000000000000.index    ← sparse offset → byte position index
  00000000000001000000.log      ← messages, segment 2 (rolled at 1GB or 1 week)
  00000000000001000000.index
```

- **Append-only writes** → sequential I/O → high throughput (disk can do 500 MB/sec sequential vs 100 IOPS random)
- **Sparse index** — index entry every N KB (e.g. 4KB); binary search index → seek to nearby position → scan forward to exact offset
- **Segments** — old segments deleted when retention expires; compacted topics keep only latest value per key

**Message format (per entry in .log):**

```
[offset: int64][length: int32][CRC: int32][magic: int8]
[attributes: int8][timestamp: int64][key: bytes][value: bytes]
```

### Replication (ISR Model)

Each partition has 1 **leader** and N-1 **followers** (In-Sync Replicas, ISR).

```
Producer → Leader
              │ append to local log
              │ followers fetch from leader (pull-based replication)
              │
              │ acks=all: wait for all ISR to confirm
              │ acks=1:   return after leader write
              │ acks=0:   fire and forget
              ▼
         return offset to producer
```

- **ISR** — set of replicas within a configurable lag threshold (e.g. 10s). Slow replicas are dropped from ISR.
- **Leader election** — if leader crashes, controller (ZooKeeper / KRaft) picks a new leader from ISR. Non-ISR replicas are ineligible (would lose data).
- **Unclean leader election** — off by default; allows out-of-ISR replica to become leader, risking data loss, in exchange for availability.

### Consumer Group & Partition Assignment

```
Consumer Group "order-service" subscribes to topic "orders" (4 partitions):

  C1 → P0, P1
  C2 → P2, P3

If C2 crashes → rebalance:
  C1 → P0, P1, P2, P3
```

- **Group Coordinator** — one broker per group; tracks members and triggers rebalance.
- **Partition assignment strategies:**
  - Range — contiguous partition blocks per consumer
  - Round-robin — interleaved; better balance
  - Sticky — minimize partition movement on rebalance

### Offset Management

Consumer group commits offsets to the internal `__consumer_offsets` topic (itself a partitioned, replicated log).

```
commit offset flow:
  consumer processes message at offset 42
  → commitOffset(topic="orders", partition=0, group="order-service", offset=43)
  → broker writes to __consumer_offsets
  → on consumer restart, fetch last committed offset → resume from 43
```

**Auto-commit vs manual commit:**

| | Auto-commit | Manual commit |
|---|---|---|
| Risk | Can commit before processing → at-most-once on crash | Must commit after processing → at-least-once |
| Control | None | Full |
| Default | Kafka default (every 5s) — risky for critical pipelines |

### Producer Idempotency & Exactly-Once

```
Idempotent producer:
  Each producer gets a PID (producer ID) from broker.
  Each message tagged with (PID, sequence_number).
  Broker deduplicates retries within a session window.

Transactional (exactly-once):
  producer.beginTransaction()
  producer.send(topic, msg)
  consumer.commitOffset(...)   ← atomic with the send
  producer.commitTransaction()
  → broker makes both visible atomically; consumer only sees committed data
```

### Push Subscription (Google Pub/Sub style)

For push delivery, a **Push Server** component reads from partitions and HTTP POSTs to subscriber endpoints:

```
Push Server
  └─ polls partition (pull internally)
  └─ HTTP POST message to subscriber URL
  └─ on 2xx → commit offset
  └─ on 5xx / timeout → retry with exponential backoff → DLQ after N failures
```

---

## Failure Scenarios

| Failure | Impact | Mitigation |
|---|---|---|
| Broker crash | Partitions led by that broker unavailable | ISR election; new leader in < 10s |
| Network partition | Follower falls behind | Dropped from ISR; rejoins after catching up |
| Consumer crash | In-flight messages not acked | Resume from last committed offset; re-process |
| Slow consumer | Falls behind retention window | Messages expire; consumer must seek forward or rebuild state |
| Poison pill message | Consumer crashes on every attempt | DLQ after N retries; skip + log |
| ZooKeeper outage | No new leader elections | Kafka KRaft removes ZK dependency; quorum-based internally |

---

## Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Storage | Segmented append-only log | Sequential I/O; retention + replay for free |
| Replication | ISR pull-based | Followers control their own fetch rate; no head-of-line blocking |
| Consumer model | Pull + offset tracking | Backpressure; replay; consumers are independent |
| Delivery | At-least-once default; EOS optional | Simple default; exactly-once available when needed |
| Offset storage | Internal topic (`__consumer_offsets`) | Leverages same replication machinery; no external DB |
| Leader election | KRaft (Raft consensus) | Removes ZooKeeper dependency; faster failover |
| Ordering | Per-partition strict; cross-partition best-effort | Key-based routing preserves order for related events |
| Fan-out | Consumer groups | Independent offset per group; zero broker-side fan-out cost |

---

## Summary Cheatsheet

```
Topic → N Partitions → each partition = append-only log on 1 leader + K followers

Write path:  producer → leader → ISR followers → ack → return offset
Read path:   consumer → fetch(partition, offset) → batch of messages → commit offset

Fan-out:     each consumer GROUP gets its own copy of every message
Replay:      seek to any offset within retention window
Scale:       add partitions → more parallelism; add brokers → more storage + I/O
```
