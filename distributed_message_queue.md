# Distributed Message Queue

**Examples:** Amazon SQS, RabbitMQ, Kafka (log-based)

**Core value:** Decoupling, async processing, buffering, retries, dead-letter queues

---

## Concepts

### Ordering

| Mode | How | Tradeoff |
|---|---|---|
| **Best-effort** | Messages delivered roughly in order; network delays can reorder | Default in most queues; higher throughput |
| **Strict (FIFO)** | Producer attaches sequence number or server assigns monotonic counter; consumer reorders before processing | Lower throughput; head-of-line blocking — a delayed message blocks all behind it |

---

# SQS-like Queue — System Design

---

## Functional Requirements

- `createQueue(name, config)` → queueURL
- `sendMessage(queueURL, body, delaySeconds?)` → messageId
- `receiveMessage(queueURL, maxMessages, visibilityTimeout)` → `[]{messageId, body, receiptHandle}`
- `deleteMessage(queueURL, receiptHandle)` — ack after processing
- `deleteQueue(queueURL)`
- Dead-letter queue (DLQ) — move messages that fail N times

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Durability | No message loss after `sendMessage` returns |
| Availability | 99.99% |
| Throughput | 100K msg/sec per queue |
| Latency | Send < 5ms p99, receive < 10ms p99 |
| At-least-once delivery | Message delivered ≥ 1 time; consumer must be idempotent |
| Scalability | Queues and throughput scale horizontally |

---

## Capacity Estimation

**Assumptions:** 1M producers/consumers, 1B messages/day, avg message size 1 KB.

- **80/20 rule:** 80% of messages in 20% of the day → peak = (1B × 0.8) / (0.2 × 86,400) ≈ **~46K msg/sec**
- **Storage:** 1B msg/day × 1 KB = **1 TB/day**. Retain for 4 days (SQS default) → **4 TB** raw; with 3× replication → **12 TB**
- **Bandwidth:** 46K msg/sec × 1 KB = **~46 MB/sec** inbound; similar outbound

---

## High-Level Architecture

```
Producers
    │  sendMessage (HTTP/gRPC)
    ▼
[ Frontend Servers ]  ← stateless, horizontally scaled, auth + routing
    │
    ▼
[ Message Store ]  ← partitioned, replicated log on disk
    │
    ▼
[ Frontend Servers ]  ← receiveMessage (long-poll)
    │
    ▼
Consumers
    │  deleteMessage (ack)
    ▼
[ Message Store ]  ← mark message deleted / advance offset
```

**Supporting services:**
- **Metadata Service** — stores queue configs (name → partitions, DLQ, visibility timeout, retention)
- **In-flight Tracker** — tracks messages currently checked out (visibility timeout); re-enqueues on timeout
- **ZooKeeper / etcd** — partition-to-node mapping, leader election
- **DLQ** — a regular queue where messages are moved after `maxReceiveCount` failures

---

## Low-Level Design

### Data Model

**Queue metadata** (stored in Metadata Service / DB):

```
Queue {
  queueId     : UUID
  name        : string
  partitions  : int
  visibilityTimeoutSec : int   // default 30s
  retentionSec         : int   // default 4 days
  maxReceiveCount      : int   // before DLQ
  dlqId                : UUID?
}
```

**Message** (stored on partition nodes):

```
Message {
  messageId    : UUID (snowflake)
  queueId      : UUID
  partitionId  : int
  offset       : int64         // monotonic within partition
  body         : bytes
  attributes   : map<string,string>
  enqueueTime  : int64 (ms)
  delayUntil   : int64 (ms)
  receiveCount : int
  status       : AVAILABLE | IN_FLIGHT | DELETED
}
```

**In-flight record** (stored in Redis / in-memory with TTL):

```
InFlight {
  receiptHandle : UUID         // opaque token given to consumer
  messageId     : UUID
  partitionId   : int
  offset        : int64
  visibleAt     : int64 (ms)   // when to re-enqueue if not acked
}
```

---

### Partitioning

Each queue is split into **N partitions** (default: 4, configurable). Each partition is an ordered log on one node.

```
sendMessage:
  partitionId = hash(messageGroupId ?? random) % N
  append to partition log

receiveMessage:
  round-robin across partitions (or consumer group assignment)
  return up to maxMessages from available offsets
```

- Partitions enable horizontal throughput scaling.
- Messages within a partition are ordered; cross-partition ordering is best-effort.
- For **FIFO queues**, use `messageGroupId` — all messages with the same group go to the same partition, preserving order within a group.

---

### Message Lifecycle

```
sendMessage
    │
    ▼
AVAILABLE (after delayUntil passes)
    │
    │ receiveMessage → sets visibilityTimeout
    ▼
IN_FLIGHT
    │               │
    │ deleteMessage  │ visibilityTimeout expires
    ▼               ▼
DELETED          AVAILABLE again (re-enqueue)
                    │
                    │ receiveCount > maxReceiveCount
                    ▼
                 → DLQ
```

---

### Visibility Timeout (the "lease" model)

SQS does not use explicit acks at the protocol level. Instead:

1. Consumer calls `receiveMessage` → gets message + `receiptHandle` + starts a timer (`visibilityTimeout`, e.g. 30s).
2. Message is hidden from other consumers for that duration.
3. Consumer processes and calls `deleteMessage(receiptHandle)` → message permanently removed.
4. If consumer crashes or times out → message becomes visible again → redelivered.

**Extending leases:** Consumer can call `changeMessageVisibility(receiptHandle, newTimeout)` for long-running jobs.

---

### Storage Layer

Each partition node stores messages as a **write-ahead log (WAL)** on disk:

```
partition_0.log:
  [offset=1][len=512][body...][checksum]
  [offset=2][len=1024][body...][checksum]
  ...
```

- Append-only → sequential writes → high throughput
- Index file maps `offset → byte position` for O(1) seeks
- Deletion is logical (mark + compaction); physical GC runs on retention expiry

**Replication:** Each partition has 1 leader + 2 followers (ISR — in-sync replicas). Leader handles all reads/writes. Followers replicate asynchronously. `sendMessage` returns success after leader write + 1 follower ACK (semi-sync).

---

### Replication & Durability

```
Producer → Leader (partition node)
              │ write to WAL
              │ replicate to Follower 1, Follower 2
              │ wait for ≥1 follower ACK
              ▼
         return messageId to producer
```

| Strategy | Durability | Latency |
|---|---|---|
| Sync (wait all replicas) | Strongest | Highest |
| Semi-sync (wait ≥1) | Strong | Medium — **recommended** |
| Async (no wait) | Weakest (potential loss) | Lowest |

---

### Receiving Messages (Long Polling)

```
Consumer → receiveMessage(queueURL, maxMessages=10, waitTimeSeconds=20)
              │
              ├─ check partition for AVAILABLE messages
              │   ├─ found → return immediately
              │   └─ none → hold connection open up to waitTimeSeconds
              │              └─ wake on new message or timeout
              │
              └─ mark returned messages IN_FLIGHT (set visibilityTimeout in Redis)
```

Long polling reduces empty responses and saves cost vs short polling.

---

### Dead-Letter Queue (DLQ)

```
receiveCount >= maxReceiveCount?
    └─ YES → move message to DLQ (copy body, add metadata: originalQueue, failureReason)
           → delete from source queue
```

- DLQ is a regular queue — can be inspected, replayed, alarmed on.
- Operators replay DLQ by re-sending messages back to the source queue after fixing the bug.

---

### Delay Queues

`sendMessage(body, delaySeconds=60)` → sets `delayUntil = now + 60s`.

Partition node skips messages where `now < delayUntil` during `receiveMessage` scans. A background sweeper periodically promotes delayed messages to AVAILABLE.

---

## Failure Scenarios

| Failure | Impact | Mitigation |
|---|---|---|
| Partition leader crash | That partition unavailable | ISR election; follower promoted in < 10s |
| Consumer crash mid-processing | Message stuck IN_FLIGHT | visibilityTimeout expires → redelivered |
| In-flight tracker (Redis) crash | All timeouts lost → messages stuck IN_FLIGHT | Redis replica + AOF; fallback: scan for old IN_FLIGHT records on restart |
| Network partition (split-brain) | Two leaders for same partition | Require quorum write; minority leader rejects writes |
| Poison pill message | Repeatedly fails, blocks processing | DLQ after maxReceiveCount; consumer must handle idempotently |

---

## Summary Cheatsheet

| Decision | Choice | Reason |
|---|---|---|
| Storage | Append-only WAL per partition | Sequential writes = high throughput |
| Replication | Leader + 2 followers, semi-sync | Durability without full sync latency |
| Delivery | At-least-once via visibility timeout | Simple; consumer idempotency handles duplicates |
| Ordering | Per-partition FIFO; FIFO queues use messageGroupId | Balance between throughput and ordering |
| Scaling | Add partitions / nodes | Stateless frontend; data layer scales horizontally |
| Failure recovery | ISR election + visibility timeout re-enqueue | No single point of failure |
| Delayed messages | `delayUntil` field + background sweeper | No separate queue needed |
| Failed messages | DLQ after N failures | Isolates poison pills from healthy traffic |
