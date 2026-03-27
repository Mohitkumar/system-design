# Coordination Services (ZooKeeper / etcd)

## What They Provide

Built on consensus (ZAB / Raft), offering strongly consistent primitives that are hard to build correctly from scratch:

| Feature | Description |
|---------|-------------|
| Linearizable atomic operations | Compare-and-set, test-and-set — exactly-once leader election |
| Total ordering of operations | Fencing tokens (monotonically increasing `zxid` / `revision`) |
| Failure detection | Sessions + heartbeats; ephemeral nodes/leases deleted on session expiry |
| Change notifications | Clients notified when a key changes (watch) |
| Service discovery | Register service instances; watch for changes |
| Work allocation | Distribute tasks across workers; detect dead workers |

---

## Data Models

| | ZooKeeper (ZAB) | etcd (Raft) |
|--|---|---|
| Model | Hierarchical **znodes** (like a filesystem) | Flat **key-value** store |
| Node types | Persistent, Ephemeral, Sequential, Ephemeral-Sequential | Keys with optional TTL lease |
| Watch model | **One-shot** per watch — fires once, must re-register | **Continuous stream** — single watch streams all future changes |
| Transactions | `multi()` — atomic batch of ops | `Txn` — if/then/else CAS batch |
| Versioning | `version` (update count), `czxid`/`mzxid` (creation/modify zxid) | Global `revision` (monotonically increasing per cluster write) |

---

## Feature 1: Leader Election

### ZooKeeper (ZAB)
```
All candidates try to create: /election/leader  (EPHEMERAL node)

Only one CREATE succeeds (linearizable — ZAB ensures one writer)
  → Winner = leader

Losers: watch /election/leader for NodeDeleted event
  → On leader crash: ephemeral node auto-deleted (session expires ~30s after crash)
  → All watchers notified → race to CREATE again → new leader elected

Fencing: leader's session ID acts as epoch; any write from old session ID rejected
         after session expiry (zombie leader cannot re-create an ephemeral node)
```

### etcd (Raft)
```
Use etcd election API (built on leases):

1. Leader creates lease:    PUT /election/leader  value=nodeA  lease=12345  (TTL=10s)
   - Succeeds only if key doesn't exist (prevExist=false) → CAS

2. Leader heartbeats lease: KeepAlive(lease=12345) every 3s → extends TTL

3. Leader crashes: lease expires after 10s → key auto-deleted

4. Candidates watch /election/leader → on delete → retry PUT with prevExist=false
   → One wins (Raft serializes all puts) → new leader

Fencing token: lease ID (monotonically increasing) → storage backends check lease ID
               to reject writes from expired (zombie) leaders
```

**Key difference**: ZooKeeper uses session expiry (TCP keepalives); etcd uses explicit leases with TTL. etcd TTL is more predictable; ZooKeeper session timeout depends on network conditions.

---

## Feature 2: Distributed Lock

### ZooKeeper (ZAB) — Herd-Avoiding Lock
```
Acquire:
  1. Create EPHEMERAL SEQUENTIAL node: /locks/lock-0000000042
  2. List /locks/ children → get all sequence numbers
  3. If my node has the lowest number → I hold the lock
  4. Else → watch the predecessor node (lock-0000000041), not all nodes
             (avoids herd effect — only 1 waiter woken per unlock)

Release:
  - Delete my ephemeral sequential node
  - Predecessor's watcher fires → that client checks if it now has lowest → acquires lock

Crash safety:
  - Client crashes → session expires → ephemeral node auto-deleted → next waiter woken
  - Fencing token = sequence number (monotonically increasing across all locks)
```

### etcd (Raft) — Lease-Based Lock
```
Acquire:
  1. Create lease: leaseid = Grant(TTL=30s)
  2. CAS: Txn{ if /locks/mylock doesn't exist → PUT /locks/mylock=clientA with leaseid }
  3. If Txn fails → watch /locks/mylock → retry on deletion

Heartbeat:
  - KeepAlive(leaseid) every 10s while holding lock

Release:
  - DELETE /locks/mylock  OR  let lease expire (crash recovery)

Fencing:
  - Pass leaseid to downstream services; they verify lease still valid via etcd before acting
  - Storage: `if request.leaseID == current_lock_leaseID → accept write`
```

**Why fencing tokens matter**:
```
T1: Client A holds lock, leaseid=5. A gets GC pause for 40s.
T2: Lock expires. Client B acquires lock, leaseid=6.
T3: A wakes up, still thinks it holds lock → tries to write with leaseid=5
    Storage sees leaseid=5 < current=6 → REJECT → split-brain prevented
```

---

## Feature 3: Failure Detection / Ephemeral Nodes

### ZooKeeper (ZAB)
```
Client opens session → negotiates sessionTimeout (e.g., 30s)
Client sends heartbeats every sessionTimeout/3 (~10s)

If ZooKeeper doesn't hear from client for sessionTimeout:
  → Session declared expired
  → All EPHEMERAL nodes created by that session → auto-deleted
  → All watches on those nodes → fire NodeDeleted event

Use case: service registry
  Service A creates /services/api/A (EPHEMERAL)
  A crashes → after 30s, /services/api/A deleted → load balancer watch fires → removes A
```

### etcd (Raft) — Leases
```
Client creates lease: leaseid = Grant(TTL=15s)
Client attaches keys to lease: PUT /services/api/A  lease=leaseid
Client sends KeepAlive every 5s

If KeepAlive stops:
  → After TTL expires, lease revoked
  → All keys attached to lease → auto-deleted
  → Watch on those keys → fires DELETE event

Advantage over ZooKeeper: lease TTL is explicit and tunable per key;
ZooKeeper session timeout is per-connection (all ephemeral nodes share one timeout)
```

---

## Feature 4: Watch / Change Notifications

### ZooKeeper (ZAB) — One-Shot Watch
```
Client: getData("/config/timeout", watch=true)
Server: returns current value + registers one-shot watch

Config changes:
  → Server sends WatchedEvent(NodeDataChanged, /config/timeout) to client
  → Watch is consumed (one-shot) → client must re-register watch

Gotcha: between event delivery and re-registration, changes can be missed
  → Client must re-read data + re-register atomically:
    getData("/config/timeout", watch=true)  ← atomic read+register
```

### etcd (Raft) — Continuous Stream Watch
```
Client: Watch("/config/", prefix=true)  ← watches entire subtree
Server: returns a stream; keeps sending events indefinitely

Event types: PUT (create/update), DELETE
Each event carries: key, value, prev_value, revision

Client processes events sequentially; no re-registration needed
On disconnect: resume with Watch("/config/", start_revision=lastSeen+1)
               → no events missed (etcd keeps compacted history)

Used by Kubernetes: controller-manager watches /registry/pods/ stream
  → reacts to every pod create/update/delete in real time
```

---

## Feature 5: Service Discovery

### ZooKeeper (ZAB)
```
Registration (on service start):
  Create EPHEMERAL node: /services/payments/10.0.1.5:8080
  Node data: {"version":"2.1","region":"us-east","weight":100}

Discovery (load balancer / client):
  getChildren("/services/payments", watch=true)
  → Returns ["10.0.1.5:8080", "10.0.1.6:8080"]
  → On NodeChildrenChanged event → refresh list

Deregistration:
  Explicit: delete node on graceful shutdown
  Implicit: crash → session expires → ephemeral node deleted → watch fires

Used by: Kafka (broker list in /brokers/ids/), HBase (RegionServer list)
```

### etcd (Raft)
```
Registration:
  leaseid = Grant(TTL=15s)
  PUT /services/payments/10.0.1.5:8080  value=<metadata>  lease=leaseid
  KeepAlive(leaseid) every 5s

Discovery:
  Watch("/services/payments/", prefix=true)
  → Stream of PUT/DELETE events as instances come and go

Used by: Kubernetes endpoints controller, Consul (uses Raft internally),
         CoreDNS (reads service records from etcd)
```

---

## Feature 6: Configuration Management

### ZooKeeper (ZAB)
```
Store config at: /config/feature_flags  (PERSISTENT node)

All services: getData("/config/feature_flags", watch=true)
  → cache value locally
  → on NodeDataChanged → re-read → update local cache

Update: setData("/config/feature_flags", newValue)
  → ZAB broadcasts to all ZooKeeper nodes → consistent across cluster
  → all watchers notified

Versioning: each setData increments node's `version` field
            CAS update: setData(path, value, expectedVersion) → fails if version mismatch
```

### etcd (Raft)
```
Store: PUT /config/feature_flags  value=<json>

All services: Watch("/config/feature_flags")
  → stream of updates; each carries revision number

CAS update:
  Txn{
    if /config/feature_flags.mod_revision == 42  →  PUT /config/feature_flags = newValue
    else → no-op (someone else updated it)
  }

History: etcd retains old revisions until compacted
  GET /config/feature_flags?revision=100  → value as of revision 100
  Useful for: audit log, rollback, debugging config drift
```

---

## ZooKeeper vs etcd — Detailed Comparison

| | ZooKeeper (ZAB) | etcd (Raft) |
|--|---|---|
| Consensus | ZAB (epoch-based, explicit sync phase) | Raft (term-based, log-first) |
| API | Custom binary protocol | gRPC + HTTP/2 |
| Data model | Hierarchical znodes (like inode tree) | Flat KV with range queries |
| Watch | One-shot, re-register required | Continuous stream, resume by revision |
| Lease/Session | Per-connection session (TCP keepalive) | Explicit per-key lease (TTL) |
| Transactions | `multi()` — atomic, no conditions | `Txn` — if/then/else with CAS |
| Read linearizability | `sync()` call required for linearizable read | `serializable` vs `linearizable` read option |
| Write throughput | ~10k–50k ops/s (Java GC impact) | ~10k–100k ops/s (Go, lower GC jitter) |
| Typical cluster size | 3 or 5 nodes | 3 or 5 nodes |
| Used by | Kafka, HBase, Storm, Hadoop HDFS HA | Kubernetes, CockroachDB, TiKV (PD), Consul |
| Operational complexity | Higher (JVM tuning, GC pauses affect sessions) | Lower (single Go binary) |

---

## Usually Used Indirectly

| System | Uses | For |
|--------|------|-----|
| Kafka (pre-KRaft) | ZooKeeper | Controller election, broker registry, topic metadata |
| Kafka KRaft | Built-in Raft | Replaced ZooKeeper entirely (Kafka 3.3+) |
| Kubernetes | etcd | All cluster state: pods, services, configmaps, secrets |
| HBase | ZooKeeper | Master election, RegionServer liveness, root region location |
| HDFS HA | ZooKeeper | NameNode active/standby failover |
| TiKV / TiDB | etcd (via PD) | Region metadata, TSO, placement scheduling |
| CockroachDB | Built-in Raft | Per-range consensus; no external coordinator needed |

---

## Key Principle

> Use ZooKeeper/etcd for coordination — not user data. They are optimized for small, highly consistent metadata.

- Typical payload: bytes to KB per key (not MB)
- Write throughput: ~10k–100k ops/s (consensus cost per write)
- Read throughput: high (followers or cached); writes serialized through leader
- Cluster size: always odd (3, 5, 7) — majority quorum required
