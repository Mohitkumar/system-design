# Multi-Raft

## Problem: One Raft Group Doesn't Scale

Single Raft group → single leader → all writes serialized through one node → throughput ceiling.

---

## Multi-Raft

Run **many independent Raft groups in parallel** — one per data shard (Range in CockroachDB, Region in TiKV).

```
Node 1          Node 2          Node 3
┌──────────┐   ┌──────────┐   ┌──────────┐
│ Region A │   │ Region A │   │ Region A │  ← Raft group A
│ Region B │   │ Region B │   │ Region B │  ← Raft group B
│ Region C │   │ Region C │   │ Region C │  ← Raft group C
└──────────┘   └──────────┘   └──────────┘
Each node hosts hundreds of Raft groups simultaneously
```

- Each region/range = its own independent Raft consensus group (typically 3 replicas)
- Writes to different regions proceed in parallel — no global serialization
- Each region has its own leader (may be on different nodes)
- Adding nodes → more regions distributed → linear write scalability

---

## Challenges

### 1. Heartbeat Storm
- Naive: every Raft group sends heartbeats independently → O(regions × nodes) messages/sec
- 1000 regions × 3 nodes = 3000 heartbeats/sec just for liveness

**Fix — Coalesced Heartbeats (CockroachDB)**:
```
Instead of: Region1→Node2, Region2→Node2, Region3→Node2 (separate msgs)
Send:       Node1→Node2 { region1: ok, region2: ok, region3: ok } (one msg)
```
Reduces heartbeat traffic from O(regions) to O(nodes).

### 2. Leader Placement
- Leaders should be co-located with the node serving client requests
- Leaders scattered randomly → extra hops

**Fix**: Placement Driver (TiKV) / Rebalancer (CockroachDB) actively moves leaders to balance load and minimize cross-node RPCs.

### 3. Snapshot Overhead
- New replica or lagging replica needs a full snapshot
- Naive: snapshot per region separately → bursty I/O

**Fix**: Rate-limit snapshot transfers; use RocksDB `IngestExternalFile` for fast atomic snapshot application (TiKV).

### 4. Linearizable Reads Without Log Writes
- Naive: every read goes through Raft log → write amplification just for reads

**Fix — Read Index (Raft optimization)**:
1. Leader records current commit index
2. Sends heartbeat to confirm it's still the leader (majority ACK)
3. Once responses received, serve read locally — no log entry needed
4. Guarantees: read sees all writes committed before the read started

---

## Region / Range Sizing

| System | Unit | Default Size | Split Trigger |
|--------|------|-------------|---------------|
| TiKV | Region | 96 MB | Size exceeds threshold |
| CockroachDB | Range | 512 MB | Size OR QPS threshold |

**Range-based (not hash-based)**:
- Range scans stay within one or few contiguous regions
- Split/merge is purely metadata — no data movement
- Tradeoff: sequential writes (e.g., monotonic IDs) can hot-spot one region

---

## Region Split / Merge

### Split
1. Leader detects region too large or too hot
2. Finds split key (middle key or hot boundary)
3. Proposes split via Raft
4. Parent region → two child regions; update metadata (PD/meta ranges)
5. Two independent Raft groups from here on

### Merge
1. Two adjacent small/cold regions identified
2. Source region's Raft group dissolved
3. Key range absorbed by target region's Raft group
4. Metadata updated

---

## Placement Driver (TiKV) / Rebalancer (CockroachDB)

Centralized scheduler that orchestrates Multi-Raft cluster:

| Responsibility | How |
|----------------|-----|
| Balance replica count per node | Move replicas from over-full to under-full nodes |
| Balance leader distribution | Transfer Raft leadership |
| Trigger region split/merge | Based on size + QPS metrics reported via heartbeat |
| Enforce placement policies | Zone/rack awareness, geo-replication constraints |
| Track GC safe point | Oldest active transaction for MVCC cleanup |

TiKV PD is itself a 3-node Raft group — single point of truth with HA.
