# Coordination Services (ZooKeeper / etcd)

## What They Provide

Built on consensus (ZAB / Raft), offering:

| Feature | Description |
|---------|-------------|
| Linearizable atomic operations | Compare-and-set, test-and-set — exactly-once leader election |
| Total ordering of operations | Fencing tokens (monotonically increasing zxid / revision) |
| Failure detection | Sessions + heartbeats; ephemeral nodes deleted on session expiry |
| Change notifications (watches) | Clients notified when a key changes |
| Service discovery | Register service instances; clients watch for changes |
| Work allocation | Distribute tasks across workers; detect dead workers |

---

## Usage Patterns

### Leader Election
```
All candidates attempt to create ephemeral node /leader
→ Only one succeeds (linearizable CAS)
→ Others watch /leader for deletion
→ On deletion (leader crash) → election restarts
```

### Distributed Lock
```
Client creates ephemeral sequential node /locks/lock-0000000042
→ Check if it has the lowest sequence number
→ If yes: holds lock
→ If no: watch predecessor node; wait for deletion
→ Fencing token = sequence number (monotonically increasing)
```

### Service Discovery
```
Each service instance creates ephemeral node /services/api/instance-1
→ Load balancer watches /services/api/
→ On change: update routing table
→ On crash: ephemeral node deleted → automatically removed from routing
```

---

## ZooKeeper vs. etcd

| | ZooKeeper | etcd |
|-|-----------|------|
| Consensus | ZAB | Raft |
| API | ZooKeeper protocol | HTTP/gRPC |
| Data model | Hierarchical znodes | Flat key-value |
| Watch model | One-time per watch | Continuous stream |
| Used by | Kafka, HBase, Storm, Hadoop | Kubernetes, CockroachDB, CoreOS |
| Language | Java | Go |

---

## Usually Used Indirectly

Applications rarely use ZooKeeper/etcd directly — consumed through higher-level abstractions:
- **Apache Curator** (ZooKeeper client library)
- **Kafka controller** uses ZooKeeper/KRaft for broker metadata
- **Kubernetes** uses etcd for all cluster state
- **HBase master** uses ZooKeeper for leader election

---

## Key Principle

> Use ZooKeeper/etcd for coordination primitives — don't implement consensus from scratch.

- Provides battle-tested, linearizable guarantees
- Designed for small amounts of coordination data (not user data)
- Typically: 3–7 nodes, low write throughput, optimized for reads + watches
