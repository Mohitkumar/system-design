# Distributed Task Scheduler

**Examples:** AWS Step Functions, Airflow, Celery, Temporal, Kubernetes Jobs, Quartz, LinkedIn Azkaban

---

## Concepts

### Task Types
| Type | Description | Example |
|------|-------------|---------|
| **One-time** | Execute once at a specific time | Send email at 9am |
| **Recurring** | Cron-based periodic execution | Daily report at midnight |
| **Chained (DAG)** | Output of task A feeds task B | ETL pipeline |
| **Delayed** | Execute after N seconds | Retry after 60s backoff |
| **Immediate** | Enqueue now, run ASAP | User-triggered background job |

### Scheduling vs Execution
```
Scheduler  — decides WHEN and WHERE to run a task
Executor   — actually runs the task code
Queue      — decouples scheduling from execution; provides backpressure
```

### Task Dependency DAG
```
         [Ingest]
        /        \
  [Transform A]  [Transform B]
        \        /
         [Merge]
            |
         [Export]

Node = task. Edge = dependency (B cannot start until A completes successfully).
Fan-out: one task triggers many. Fan-in: many must complete before one can start.
```

### Fairness vs Priority
| | Priority Queue | Fair Queuing | Weighted Fair |
|--|---|---|---|
| **Behavior** | Highest priority always first | Each tenant gets equal share | Share proportional to weight |
| **Risk** | Low-priority tasks starve forever | High-priority tenant slowed | Balanced |
| **Use** | Emergency jobs, SLA tiers | Multi-tenant equality | Paid tier vs free tier |

---

## Functional Requirements

- `createTask(taskDef, scheduleAt?, cronExpr?, dependsOn[]?)` → taskId
- `cancelTask(taskId)` — cancel pending or running task
- `getTaskStatus(taskId)` → status, attempts, logs, result
- `retryTask(taskId)` — manual retry of failed task
- At-least-once execution; idempotency key for exactly-once semantics
- Cron scheduling (`0 9 * * 1-5` = 9am weekdays)
- Dependency DAGs — task starts only when all upstream tasks succeed
- Priority levels (CRITICAL / HIGH / NORMAL / LOW / BACKGROUND)
- Per-tenant isolation — one tenant's storm doesn't affect others
- Resource limits per task (CPU, memory, timeout)

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Schedule accuracy | ± 1s of scheduled time |
| Throughput | 1M tasks/sec dispatch |
| Availability | 99.99% for scheduler; tasks retry on executor failure |
| Durability | No task lost after `createTask` returns 200 |
| Scalability | 10B tasks/day, 100K concurrent running tasks |
| Latency | Task starts within 5s of schedule time |

---

## Capacity Estimation

**Assumptions:** 100K tenants, 10B tasks/day, avg task size 1 KB metadata, 100K concurrent workers.

**Task throughput:**
```
80:20 rule: avg = 10B / 86400 = ~116K tasks/sec; peak = 116K × 4 = ~460K tasks/sec
Queue dispatch rate: 460K tasks/sec → need high-throughput queue (Kafka-backed)
```

**Storage (task metadata — DynamoDB/Cassandra):**
```
Active tasks (30-day window): 10B × 30 = 300B tasks
Per task record: 1 KB (status, schedule, result, attempts)
Total: 300 TB → shard across 300 DynamoDB partitions (1 TB/partition)
Index needed: by (tenant_id, status), by (scheduled_at), by (dag_id)
```

**Scheduled task scan:**
```
Scheduler must find tasks due in next 1s: 460K tasks/sec
Polling DB at 1s interval: SELECT WHERE scheduled_at <= now() + 1s
Index on scheduled_at: sorted scan → O(tasks due) not O(total tasks)
Partitioned by scheduled_at bucket (1-minute buckets): 460K tasks × 60s = 27M rows per bucket → manageable
```

**IOPS:**
```
Task create: 460K writes/sec → DynamoDB 460K WCU (auto-scaled; ~$0.00065/WCU = $300/sec — use batching)
Scheduler poll: 460K reads/sec → DynamoDB 460K RCU (use GSI on scheduled_at)
Status update (on complete/fail): 460K writes/sec
Total DB IOPS: ~1.4M ops/sec → DynamoDB auto-scaling handles; Cassandra needs ~30 nodes

Queue IOPS:
  Kafka: 460K msgs/sec × 1 KB = 460 MB/sec → 5 brokers × 100 MB/sec = sufficient
  Sequential writes → NVMe handles easily
```

**Server count:**

| Role | Count | Sizing |
|---|---|---|
| Scheduler nodes | 10 | 16 cores, 32 GB RAM |
| Kafka brokers | 15 | 16 cores, 64 GB RAM, 2× 2TB NVMe |
| Worker nodes | 1000+ | varies (task-specific: 4–32 cores) |
| Task DB (Cassandra) | 30 | 16 cores, 128 GB RAM, 2× 2TB NVMe |
| DAG state store | 5 (etcd/Redis) | 8 cores, 32 GB RAM |
| API / gateway | 10 | 8 cores, 16 GB RAM |
| Rate limiter (Redis) | 3 | 8 cores, 32 GB RAM |

---

## High-Level Architecture

```
Client / Cron trigger / Event
          │
          ▼
    [API Gateway]
    rate limiter (per tenant)
    ID generator (Snowflake)
          │
          ▼
    [Task Store]  ──────────────────────────────────► [DAG State Store]
    (DynamoDB/Cassandra)                               (Redis / etcd)
    task metadata, status                              DAG topology, dependency state
          │
          ▼
    [Scheduler]  (polls task store for due tasks)
          │
          ├── priority queue selection
          ├── tenant fairness check
          ├── resource availability check ──► [Resource Manager]
          │
          ▼
  [Dispatch Queue] (Kafka, partitioned by tenant+priority)
    ┌────┬────┬────┬────────────────────┐
    │ P0 │ P1 │ P2 │ tenant-fair queues │
    └────┴────┴────┴────────────────────┘
          │
          ▼
    [Worker Pool]
    worker fetches task → executes → reports result
          │
          ▼
    [Task Store] update status (RUNNING → SUCCESS/FAILED)
          │
          ▼
    [DAG Engine] check if downstream tasks are unblocked → enqueue them
```

---

## Component Deep Dive

### 1. API Layer + ID Generation

```
createTask request:
  1. Rate limit: check per-tenant rate (Redis sliding window counter)
     → reject with 429 if exceeded
  2. Validate: cron expression, dependency IDs exist, resource limits within quota
  3. Generate task ID: Snowflake ID (timestamp + worker + seq) → sortable, globally unique
  4. Persist to Task Store: status=PENDING, scheduled_at, tenant_id, priority, payload
  5. If DAG task: update DAG State Store with dependency edges
  6. Return taskId to client

Idempotency:
  Client passes idempotency_key → stored in Task Store
  Duplicate createTask with same key → return existing task (no double-create)
```

### 2. Task Store Schema

**Primary table** (Cassandra / DynamoDB):
```
task_id          UUID (Snowflake)       PK
tenant_id        string
status           enum PENDING|RUNNING|SUCCESS|FAILED|CANCELLED
priority         int (0=CRITICAL, 4=BACKGROUND)
scheduled_at     timestamp
created_at       timestamp
attempts         int
max_attempts     int
timeout_secs     int
payload          bytes (task definition, up to 256 KB)
result           bytes (output or error, up to 1 MB)
worker_id        string (assigned worker)
idempotency_key  string (unique per tenant)
dag_id           UUID (null if standalone)
```

**GSIs / secondary indexes:**
```
GSI 1: (scheduled_at, status=PENDING)  → Scheduler polls this to find due tasks
GSI 2: (tenant_id, status)             → List tasks per tenant
GSI 3: (dag_id)                        → Find all tasks in a DAG
GSI 4: (idempotency_key, tenant_id)    → Dedup check on createTask
```

### 3. Scheduler

Core loop: poll → select → dispatch → track.

```
Every 1 second per scheduler shard:

1. POLL: SELECT tasks WHERE scheduled_at <= now()+1s AND status=PENDING
         AND assigned_scheduler = my_shard_id
         LIMIT 10000

2. FAIRNESS: group by tenant_id → apply weighted fair queuing
   tenant_share = task_count_from_tenant / total_pending
   if tenant_share > tenant_weight → defer some of their tasks

3. PRIORITY: within fairness budget, order by priority DESC
   CRITICAL tasks always dispatched first within tenant's share

4. RESOURCE CHECK: does cluster have capacity for this task's resource request?
   if no capacity → defer (do not dispatch)

5. CLAIM: UPDATE task SET status=DISPATCHING, scheduler_id=me WHERE status=PENDING
   (optimistic lock — CAS on status field; prevents double-dispatch)

6. DISPATCH: produce to Kafka partition = hash(tenant_id + priority_bucket)
   Message: {task_id, payload, worker_constraints, timeout}

7. WATCH: track in-flight tasks; if no heartbeat from worker after timeout → re-enqueue
```

**Scheduler sharding** — multiple scheduler nodes, each owns a key range of task IDs:
```
Scheduler 0: task_id hash 0–25%
Scheduler 1: task_id hash 25–50%
...
Avoids single-scheduler bottleneck; assignment stored in etcd
```

### 4. Priority Queue + Tenant Fairness

**Multi-level Kafka topics:**
```
Kafka topics:
  tasks.priority.critical   (1 partition per major tenant; preempts all)
  tasks.priority.high
  tasks.priority.normal
  tasks.priority.low
  tasks.priority.background

Workers subscribe to all topics with weighted polling:
  Poll critical: always check first (priority)
  Poll high:     check if critical empty
  Poll normal:   weight 0.6 of remaining capacity
  Poll low:      weight 0.3
  Poll bg:       weight 0.1

Result: background tasks never starve (always get 10% of capacity)
        critical tasks never wait (always polled first)
```

**Per-tenant fair queuing:**
```
Kafka partition key = tenant_id
Each tenant gets at most one partition per priority topic
Worker pulls round-robin across tenant partitions → no tenant monopolizes workers

Token bucket per tenant (Redis):
  Max dispatch rate = tenant_tier_limit (e.g., free=100/s, paid=10K/s)
  Scheduler checks before dispatching: if bucket empty → defer task
```

### 5. DAG Engine

Manages task dependencies; decides when downstream tasks become eligible.

**State per DAG (Redis hash):**
```
dag:{dag_id}:tasks        → set of all task IDs in DAG
dag:{dag_id}:deps:{task}  → set of upstream task IDs (what must complete first)
dag:{dag_id}:done         → set of completed task IDs
dag:{dag_id}:status       → RUNNING | SUCCESS | FAILED | CANCELLED
```

**On task completion:**
```
Task T completes (SUCCESS):
  1. Add T to dag:{dag_id}:done
  2. For each downstream task D where T ∈ deps[D]:
     a. Check: deps[D] ⊆ done → all dependencies satisfied?
     b. If yes → UPDATE task D SET status=PENDING, scheduled_at=now()
        → D becomes eligible for scheduling
  3. If all tasks in DAG are done → dag status = SUCCESS

Task T fails (FAILED, no retries left):
  1. dag status = FAILED
  2. Cancel all downstream tasks of T (they can never run — dependency failed)
  3. Leave unrelated branches running (partial execution allowed unless strict mode)

Atomicity (Redis MULTI/EXEC or Lua script):
  Check deps → update eligibility → must be atomic to prevent duplicate dispatch
```

**Fan-in example:**
```
[A] ──►
         [C]  ← C waits for both A and B
[B] ──►

deps[C] = {A, B}
done    = {A}     → C not yet eligible (B not done)
done    = {A, B}  → C eligible → enqueue C
```

### 6. Resource Manager

Tracks cluster capacity; prevents over-subscription.

```
Resource model per task:
  cpu_cores: float (0.5, 1, 4, 16)
  memory_mb: int
  gpu:       int (optional)
  timeout:   seconds

Cluster capacity (updated by worker heartbeats every 5s):
  total_cpu, total_memory, available_cpu, available_memory per worker pool

On dispatch check:
  Can this task fit on any available worker?
  if task.cpu > max_available_cpu_on_any_worker → defer

Bin packing (optional):
  Assign task to worker with least wasted resources (First Fit Decreasing)
  Avoids fragmentation (small tasks on half-full workers, large tasks can't fit)

Preemption (CRITICAL priority only):
  If no capacity and task is CRITICAL → evict a BACKGROUND task
  → re-enqueue evicted task with status=PENDING
```

### 7. Worker

Stateless executor. Pulls tasks from Kafka, runs them, reports back.

```
Worker lifecycle per task:
  1. Pull task from Kafka (at-most-once fetch; manual offset commit after ack)
  2. UPDATE task SET status=RUNNING, worker_id=me, started_at=now()
  3. Execute task (subprocess / container / lambda invocation)
  4. Heartbeat every 10s → scheduler knows task is alive
  5. On complete:
     UPDATE task SET status=SUCCESS, result=..., completed_at=now()
     Kafka offset commit (message acknowledged)
     Notify DAG engine (if task has dag_id)
  6. On failure:
     if attempts < max_attempts → UPDATE status=PENDING, scheduled_at=now()+backoff
     else → UPDATE status=FAILED
     Kafka offset commit

Timeout enforcement:
  Worker sets alarm on task start (timeout_secs)
  On alarm: kill subprocess → mark FAILED → re-enqueue if retries remain

Isolation:
  Each task runs in its own container/namespace → CPU/memory cgroup limits
  Prevents one runaway task from starving others on same worker
```

### 8. Timeout & Zombie Detection

```
Scheduler heartbeat monitor:
  Every 30s: scan task_store for tasks WHERE status=RUNNING AND heartbeat_at < now()-60s
  These are zombie tasks (worker crashed without updating status)
  → UPDATE status=PENDING, attempts+=1, scheduled_at=now()  (re-enqueue)
  → If attempts >= max_attempts → status=FAILED

Why not just rely on Kafka visibility timeout:
  Task was consumed from Kafka (offset committed) before worker crash
  Kafka does NOT know the task failed → must detect via heartbeat + DB
```

---

## Isolation Strategies

| Concern | Mechanism |
|---------|-----------|
| **Tenant CPU isolation** | Dedicated Kafka partition per tenant; per-tenant rate limit (Redis token bucket) |
| **Noisy tenant** | Max dispatch rate cap; excess tasks deferred not dropped |
| **Task isolation** | Each task in its own container (cgroups, network namespace) |
| **DAG isolation** | DAG failure does not cancel tasks in other DAGs |
| **Priority isolation** | Multi-level Kafka topics; workers check critical queue first |
| **Resource isolation** | Resource manager prevents over-subscription; hard limits via cgroups |

---

## Failure Modes

| Failure | Impact | Mitigation |
|---------|---------|------------|
| Scheduler node crash | Its task shard unscheduled temporarily | etcd lease expires → another scheduler claims the shard |
| Worker crash mid-task | Task stuck RUNNING | Heartbeat monitor detects zombie → re-enqueue after 60s |
| Kafka broker down | Dispatch pauses for affected partitions | 3 replicas; ISR election < 10s; scheduler retries produce |
| Task Store unavailable | Cannot create or update tasks | DynamoDB multi-AZ; Cassandra 3 replicas; scheduler buffers in-memory briefly |
| DAG state store (Redis) down | DAG dependency resolution pauses | Redis Sentinel; replica promotion <30s; in-flight DAGs resume on recovery |
| Duplicate dispatch | Task runs twice | Idempotency key on task; CAS on status (PENDING→DISPATCHING) prevents double-dispatch |
| Thundering herd (many tasks due at once) | Scheduler overwhelmed at midnight cron | Jitter: add random 0–60s offset to cron tasks; stagger execution |
| Poison pill task | Task always fails, re-queues forever | max_attempts cap; exponential backoff; move to DLQ after exhaustion |

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Task store | DynamoDB / Cassandra | Scale to 300B rows; GSI on scheduled_at for efficient polling |
| ID generation | Snowflake | Time-sortable, globally unique, no coordination at runtime |
| Queue | Kafka (multi-topic by priority) | Durable, replayable, high-throughput; per-tenant partitions |
| Fairness | Weighted fair queuing + per-tenant token bucket | Prevent noisy tenant; guaranteed capacity per tier |
| DAG state | Redis (in-memory, fast set operations) | Dependency check is hot path; must be fast |
| Scheduler sharding | Hash(task_id) → scheduler | Avoids single scheduler bottleneck |
| Zombie detection | Heartbeat + periodic DB scan | Worker crash cannot be detected via Kafka (offset already committed) |
| Task isolation | Container per task (cgroups) | CPU/memory limits enforced by OS; fault boundary per task |
| Dispatch atomicity | CAS on status field (PENDING → DISPATCHING) | Prevents double-dispatch even with multiple scheduler nodes |

---

## Interview Scenarios

### "Two scheduler nodes dispatch the same task simultaneously"
- Race: both see task as PENDING and produce to Kafka before either updates DB
- Fix: CAS on status — `UPDATE task SET status=DISPATCHING WHERE task_id=X AND status=PENDING`
- Only one UPDATE succeeds (DB serializes writes) → other scheduler gets 0 rows affected → skips task
- Kafka dedup: if somehow dispatched twice, worker checks task status on pickup → if already RUNNING, discard

### "DAG has 1000 tasks; upstream failure should cancel all downstream"
- Naive: recursive cancel (BFS from failed node) — can trigger thousands of DB writes
- Optimistic: mark DAG as FAILED in one write; workers check DAG status before starting each task
- On task pickup: `if dag_status == FAILED → skip task, mark CANCELLED`
- Avoids cascading writes; workers do the cleanup lazily

### "Need strict exactly-once — task must never run twice (financial debit)"
- At-least-once is default (retries on crash)
- Exactly-once requires: idempotency key in task payload + downstream service checks it
- Pattern: task writes `{idempotency_key, result}` to DB with unique constraint
  - First execution: INSERT succeeds → debit applied
  - Retry execution: INSERT fails (duplicate key) → skip debit, return cached result
- Scheduler side: prevent re-dispatch with CAS on status; worker side: idempotency key on downstream

### "Tenant submits 10M tasks at once (batch import) — starves other tenants"
- Tenant's token bucket exhausted → dispatcher defers excess tasks
- Tenant's Kafka partition fills → backpressure to API → `createTask` returns 429 for excess
- Deferred tasks stored in Task Store, re-enqueued as capacity frees up
- Other tenants unaffected: their Kafka partitions and token buckets are separate

### "High-priority task waiting behind 1M low-priority tasks"
- With single queue: FIFO means low-priority blocks high-priority
- Fix: multi-level Kafka topics — high-priority tasks dispatched from dedicated topic, always polled first
- Worker drain rate: workers check `tasks.priority.critical` before `tasks.priority.normal`
- Result: CRITICAL task dispatched within seconds regardless of normal queue depth

### "Resource fragmentation — workers have capacity but no task fits"
- Symptom: tasks queued, workers idle, but tasks need 8 cores and all workers have 3 cores free
- Fix: bin packing — consolidate tasks on fewer workers, free up whole workers for large tasks
- Preemption: evict 3 background tasks from a worker to free 8 cores for the critical large task
- Long-term: enforce max_task_cores limit at createTask time; right-size task resource requests

### "Cron job creates 100K tasks at midnight simultaneously (thundering herd)"
- All cron `0 0 * * *` tasks fire at exactly 00:00:00 → scheduler overwhelmed
- Fix: jitter — add random offset at schedule creation time: `scheduled_at = midnight + rand(0, 300s)`
- Distributes 100K tasks over 5 minutes → smooth load curve
- High-SLA cron jobs: exempt from jitter; run at exact time with dedicated scheduler shard

### "Task dependencies form a cycle (circular DAG)"
- Cycle detection at `createTask` time: DFS on dependency graph — if adding edge creates a back edge, reject
- Validate before persisting; return 400 with cycle path to caller
- Runtime guard: max depth limit (e.g., 100 hops) — prevents infinite recursion even if cycle slips through

### "Constraint: reduce cost — 90% of tasks are background batch jobs"
- Background tasks run on spot/preemptible instances (60–80% cheaper)
- Resource manager maintains two worker pools: on-demand (CRITICAL/HIGH) + spot (LOW/BACKGROUND)
- On spot eviction: task re-enqueued with attempts++ → picked up by another spot worker
- Bin packing: pack background tasks densely → fewer spot instances needed
