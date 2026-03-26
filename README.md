# System Design Notes

Sources: DDIA (Kleppmann) + Distributed Systems for Fun and Profit (Takada)

---

## concepts/

### fundamentals/
- [distributed_systems_basics.md](concepts/fundamentals/distributed_systems_basics.md) — scalability, availability, CAP, FLP, system models, safety/liveness

### time_ordering/
- [clocks_and_ordering.md](concepts/time_ordering/clocks_and_ordering.md) — physical clocks, Lamport clocks, vector clocks, failure detectors, fencing tokens
- [hybrid_logical_clock.md](concepts/time_ordering/hybrid_logical_clock.md) — HLC: causality + wall time, uncertainty windows, CockroachDB usage

### replication/
- [replication.md](concepts/replication/replication.md) — leader/follower, multi-leader, leaderless, quorum, Dynamo, CRDTs, CALM theorem

### partitioning/
- [partitioning.md](concepts/partitioning/partitioning.md) — key-range vs hash, consistent hashing, secondary indexes, rebalancing, routing

### consistency/
- [consistency_models.md](concepts/consistency/consistency_models.md) — linearizability, causal, eventual, ACID isolation levels, MVCC, snapshot isolation
- [transactions.md](concepts/consistency/transactions.md) — ACID, serial execution, 2PL, SSI
- [percolator_protocol.md](concepts/consistency/percolator_protocol.md) — Percolator 2PC, CF layout, TSO, lock resolution (TiKV/Spanner)

### consensus/
- [consensus_algorithms.md](concepts/consensus/consensus_algorithms.md) — Paxos, Raft, ZAB, total order broadcast
- [multi_raft.md](concepts/consensus/multi_raft.md) — Multi-Raft sharding, coalesced heartbeats, read-index, region split/merge, PD scheduler
- [two_phase_commit.md](concepts/consensus/two_phase_commit.md) — 2PC, distributed transactions, XA, saga pattern
- [coordination_services.md](concepts/consensus/coordination_services.md) — ZooKeeper, etcd, leader election, distributed locks

### storage/
- [storage_engine.md](concepts/storage/storage_engine.md) — B+tree, LSM tree, MVCC, indexing (clustered, secondary, hash, composite)

### conflict_resolution/
- [conflict_resolution.md](concepts/conflict_resolution/conflict_resolution.md) — vector clocks, Lamport clocks, LWW, client-side merge, CRDTs

---

## capacity/

- [capacity_planning.md](capacity/capacity_planning.md) — Power of 2, latency numbers, QPS/storage/bandwidth estimation framework, sizing cheatsheet

---

## design/

- [cockroachdb.md](design/cockroachdb.md) — distributed SQL, Raft per range, SSI, HLC, intent-based commit, DistSQL
- [tikv.md](design/tikv.md) — distributed KV, Multi-Raft regions, Percolator 2PC, TSO, RocksDB CFs
- [key_value_store.md](design/key_value_store.md)
- [distributed_cache.md](design/distributed_cache.md)
- [distributed_message_queue.md](design/distributed_message_queue.md)
- [pub_sub.md](design/pub_sub.md)
- [unique_id_generation.md](design/unique_id_generation.md)
