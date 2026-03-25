# Conflict Detection & Resolution in Distributed Systems

---

## Vector Clocks

Vector clocks are a **diagnostic tool, not a solution**. They are like a smoke detector: they tell you there's a fire (conflict), but they don't put it out. They tell you: *"Hey, Process A and Process B both edited this at the same time, and neither knew about the other."*

### How it works

Each node maintains a counter for every other node in the cluster. On every write, a node increments its own counter and attaches the full vector to the data.

```
Node A writes: [A:1, B:0]
Node B writes: [A:0, B:1]
```

To compare two clocks, every position in one must be ≤ every position in the other for it to be "older." If neither clock dominates the other, the writes are **concurrent** — a conflict.

### Writing to Different Keys

If Process A writes to `Key: "Apples"` and Process B writes to `Key: "Bananas"`, their vector clocks show concurrent writes. But since the keys are different, there is **no conflict** — the database stores both independently.

### Writing to the Same Key

If both processes write to `Key: "Inventory"`:

- Clock A is `[A:1, B:0]`
- Clock B is `[A:0, B:1]`

Neither dominates the other → **conflict detected**. The database stores both versions as "siblings" and asks the application to resolve them later.

### Example: 2-Node Cluster

| Step | Action | Clock |
|------|--------|-------|
| 1 | Node A writes `Key1=Val1` | `[A:1, B:0]` |
| 2 | Node B writes `Key2=Val2` | `[A:0, B:1]` |
| 3 | Compare clocks | `[A:1,B:0]` vs `[A:0,B:1]` → concurrent, different keys → no conflict |

### Real-World Systems Using Vector Clocks

| System | How it's used |
|--------|--------------|
| **Amazon DynamoDB** | Tracks causality between versions of the same item across replicas. Exposes conflicts to the caller via multiple "winner" versions. |
| **Riak** | Stores conflicting versions as siblings. Client reads all siblings and merges them (often via CRDTs). |
| **Voldemort** (LinkedIn) | Uses vector clocks to detect stale reads and surface conflicts. Application code performs the merge. |

---

## Lamport Clocks

Lamport clocks solve the same problem differently: they **ignore the conflict and force a winner**.

Because a Lamport clock creates a **Total Order**, there is no such thing as a "conflict" in its eyes. Every event is assigned a position in a single, straight line.

### Rules

1. Each process keeps a counter, starting at 0.
2. On any event, increment your counter by 1.
3. When sending a message, attach your current counter.
4. On receiving a message, set your counter to `max(local, received) + 1`.

### Example: Tie-Breaking

```
Client A sends write → Timestamp: 10, ProcessID: 1
Client B sends write → Timestamp: 10, ProcessID: 2

Tie-breaker: compare ProcessID → 1 < 2
Winner: Client A
Client B's write is silently overwritten.
```

### The Problem: Causally Blind

Client B may have written without ever seeing Client A's data. Lamport clocks make them look sequential even though they were simultaneous. **Client A's update is lost forever and the system never notices a conflict.**

### When to Actually Use Lamport Clocks

Use them when **agreement matters more than history**:

| Use Case | Why |
|----------|-----|
| **Distributed locks** | Two servers race for a lock — you just need one winner, not a full conflict history. |
| **Total-order multicast** | Every node must process messages in the same sequence. |
| **Leader election** | Pick one node without debate. |

### Real-World Systems Using Lamport Clocks

| System | How it's used |
|--------|--------------|
| **Apache ZooKeeper** | Uses `zxid` (Zookeeper Transaction ID), a Lamport-style monotonic counter, to totally order all transactions across the cluster. |
| **Google Chubby** | Lock service uses sequencer tokens (Lamport-style) to order lock acquisitions. |
| **Kafka** | Log offsets act as Lamport clocks — every record has a monotonically increasing offset ensuring total order within a partition. |

---

## Conflict Resolution Strategies

Once a conflict is detected (or forced), you need to resolve it. There are three main strategies:

### 1. Last Write Wins (LWW)

Ignore the vector clock warning and pick a winner using a physical timestamp.

- **Fast** — no extra logic required
- **Lossy** — one write is silently discarded

**Example — Cassandra:** By default, Cassandra uses LWW. If two clients write to the same row within milliseconds of each other, the one with the higher timestamp wins and the other write is gone. Works well for time-series data (sensor readings, logs) where the latest value is always correct by definition. Dangerous for banking or inventory counts.

---

### 2. Client-Side Merge (Sibling Resolution)

The database hands all conflicting versions ("siblings") to your application. Your code — or the user — merges them.

- **Lossless** — no data is discarded
- **Complex** — requires merge logic in the app

**Example — Riak:** A shopping cart stored in Riak can accumulate siblings when two clients add items concurrently. On the next read, Riak returns all siblings. The client merges them by taking the union of all items and writes the merged result back. This is how Amazon's original Dynamo paper described shopping cart resolution.

**Example — Git:** Every merge conflict is client-side resolution. Git detects that two branches edited the same lines and refuses to auto-merge. The developer manually resolves it, then commits the merged result.

---

### 3. CRDTs (Conflict-free Replicated Data Types)

Use **smart data structures** that are mathematically guaranteed to merge automatically and correctly, regardless of order or delivery.

- **Lossless and automatic** — no app logic or user intervention
- **Limited to specific data shapes** — not a general solution

**How it works:** A CRDT is designed so that its merge operation is commutative, associative, and idempotent. No matter which order two nodes exchange their states, the result is always the same.

| CRDT Type | Example | Merge Rule |
|-----------|---------|------------|
| G-Counter | Page view count | Sum all per-node counters |
| PN-Counter | Inventory count | Sum increments − sum decrements |
| G-Set | Unique user IDs | Union of all sets |
| LWW-Register | User profile field | Keep the one with the latest timestamp |
| OR-Set | Collaborative tags | Track add/remove pairs; removes only win if they saw the add |

**Example — Redis:** Redis Cluster uses CRDT-like logic in its `INCR` command. For geo-distributed deployments (Redis Enterprise Active-Active), each region has its own counter CRDT that automatically converges with no conflicts.

**Example — Figma:** Multiple cursors and shape edits in Figma use CRDT-like operational transforms so two designers can move the same object simultaneously and the final position converges correctly on both screens without a conflict dialog.

**Example — Apple Notes / iCloud:** iCloud syncing uses CRDTs for text editing. Offline edits on your iPhone and Mac merge automatically when both come online — no "merge conflict" prompt ever appears.
