# Rate Limiter — System Design

## Functional Requirements
- Limit requests per client (user ID, API key, IP) per time window
- Support multiple rules: per-endpoint, per-user tier, global
- Soft limit (queue) vs hard limit (reject with 429)
- Return rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Non-Functional Requirements
- **Latency**: < 5ms overhead per request (must not slow down the API)
- **Throughput**: handle millions of requests/sec across distributed nodes
- **Availability**: rate limiter failure should not block traffic (fail open)
- **Accuracy**: small over-counting acceptable; under-counting (allowing too many) is not
- Distributed — works correctly across multiple LB/API gateway nodes

---

## Capacity Estimation

```
DAU: 10M users
Avg QPS: 10M × 100 req/day / 86,400 ≈ 12k QPS
Peak QPS: 12k × 3 = 36k QPS (80:20 rule)

Redis key per user: user_id + window = ~50 bytes key
Counter value: 8 bytes
Total keys in memory (1hr window): 10M × 58 bytes ≈ 580 MB  ← fits in single Redis node

IOPS:
  Every request = 1 Redis INCR (atomic) + 1 EXPIRE (on first request in window)
  Peak: 36k req/s → 36k Redis ops/s
  Redis is memory-only → 0 disk IOPS for counter ops
  AOF persistence (fsync=every-sec): 1 batch disk write/sec — negligible
  Rule cache lookup: in-process (0 disk IOPS); DB refresh every 60s = ~1 DB read/min

Server count:
  Redis (counter store): 1 primary handles 36k ops/s easily (Redis = 100k+ ops/s)
                         Run 3 nodes (primary + 2 replicas) for HA → Redis Sentinel
  API gateway nodes:     36k req/s ÷ 10k req/server = 4 (run 6–8 with headroom)
  Rules DB (PostgreSQL): 1 primary + 1 read replica (low QPS — config reads only)

Network overhead per request: 1 Redis round-trip ≈ 0.5–1ms (same-DC)
Total added latency: < 2ms per request (well within 5ms SLO)
```

---

## Algorithms

### 1. Token Bucket

```
bucket_capacity = 10
refill_rate     = 10 tokens/sec

On request:
  tokens += (now - last_refill) × rate
  tokens  = min(tokens, capacity)
  last_refill = now
  if tokens >= 1: tokens -= 1; allow
  else: reject (429)
```

- **Allows bursts** up to bucket capacity
- Smooth average rate enforced over time
- State: `(tokens, last_refill_time)` per key — 2 values in Redis
- **Race condition**: must update atomically (Redis Lua script)

| ✅ Pros | ❌ Cons |
|---------|---------|
| Allows burst traffic | Two values to store and update atomically |
| Memory-efficient | Clock precision matters for refill |
| Easy to tune burst vs avg rate | Burst can cause downstream spike |

**Used by**: Stripe API, AWS API Gateway, Nginx `limit_req`

---

### 2. Leaky Bucket

```
Queue requests up to queue_size
Drain queue at fixed rate (e.g., 10 req/sec)
If queue full → drop request
```

- Enforces **strict constant output rate** regardless of input bursts
- Smooths bursty input → steady output
- State: queue of pending requests per client

| ✅ Pros | ❌ Cons |
|---------|---------|
| Smooth output — protects backend | Queue adds latency |
| Simple to understand | Old requests in queue may expire before processing |
| No burst pass-through | Memory scales with queue size |

**Used by**: traffic shaping (network devices), Shopify, queued API processing

---

### 3. Fixed Window Counter

```
window_key = user_id + floor(now / window_size)
counter = INCR(window_key)
if counter == 1: EXPIRE(window_key, window_size)
if counter > limit: reject (429)
```

- Reset counter every window (1min, 1hr)
- **Edge spike problem**: 100 req in last 1s of window + 100 in first 1s of next = 200 in 2s window

```
Window:  |─────60s─────|─────60s─────|
         ........100   100..........
                   ↑   ↑
              burst of 200 in 2 seconds — window boundary attack
```

| ✅ Pros | ❌ Cons |
|---------|---------|
| Simple — 1 Redis INCR + EXPIRE | Boundary burst allows 2× limit |
| Memory-efficient (1 key per window) | Not precise |
| Fast | Resets hard — sudden quota refresh |

**Used by**: simple API rate limiting where precision is not critical

---

### 4. Sliding Window Log

```
On each request:
  ZREMRANGEBYSCORE(key, 0, now - window_size)  # remove old entries
  ZADD(key, now, request_id)                   # add current
  count = ZCARD(key)
  EXPIRE(key, window_size)
  if count > limit: reject
```

- Stores **every request timestamp** in a sorted set
- Exact count of requests in the last N seconds at any point

| ✅ Pros | ❌ Cons |
|---------|---------|
| Exact — no boundary burst | Memory = O(requests in window) per user |
| Fair sliding window | At high QPS: many ZADD entries per key |
| Precise reset timing | Expensive for high-rate users |

**Used by**: low-traffic precision APIs, financial systems, per-user strict enforcement

---

### 5. Sliding Window Counter (Recommended)

```
Approximate using two fixed windows + interpolation:

prev_count   = GET(prev_window_key)
curr_count   = GET(curr_window_key)
elapsed      = (now % window_size) / window_size  # fraction through current window

approx_count = prev_count × (1 - elapsed) + curr_count

if approx_count + 1 > limit: reject
else: INCR(curr_window_key)
```

**Example**:
```
window = 60s, limit = 100
prev_window (00:00–01:00): 80 requests
curr_window (01:00–02:00): 30 requests, elapsed = 45s into window

approx = 80 × (1 - 45/60) + 30 = 80×0.25 + 30 = 50 → allow
```

- Only ~1% error rate vs sliding window log
- 2 Redis keys per user (current + previous window)

| ✅ Pros | ❌ Cons |
|---------|---------|
| Memory-efficient (2 counters) | Approximate (~1% error) |
| Accurate enough for production | Slightly more complex than fixed window |
| No boundary burst problem | |
| Fast (2 GET + 1 INCR) | |

**Used by**: Cloudflare, most production API gateways, Nginx

---

## Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst | Best For |
|-----------|--------|----------|-------|----------|
| Token Bucket | Low | Good | ✅ Allows burst | API GW, allows burst traffic |
| Leaky Bucket | Medium | Exact output | ❌ Queues | Traffic shaping, steady downstream |
| Fixed Window | Very Low | Poor (boundary) | ✅ | Simple cases, non-critical |
| Sliding Window Log | High | Exact | ❌ | Precision needed, low traffic |
| **Sliding Window Counter** | **Low** | **~99%** | **~None** | **Production default** |

---

## Data Model

### Rule Storage

```yaml
# rules.yaml — stored in config service or DB
- api: /v1/payments
  limit: 10
  window: second
  by: user_id

- api: /v1/search
  limit: 100
  window: minute
  by: api_key

- global:
  limit: 1000
  window: minute
  by: ip
```

- Rules stored in DB (PostgreSQL) — source of truth
- Rules cached in Redis (or in-process) — fast lookup per request
- Cache TTL: 60s — tolerate slightly stale rules

### Counter Storage (Redis)

```
Key format:  rl:{user_id}:{endpoint_hash}:{window_start}
Value:       integer counter
TTL:         window_size × 2 (keep previous window too)

Example:
  rl:user_123:/v1/pay:1700000060  → 7
  TTL: 120s
```

---

## Architecture

```
Client Request
     │
     ▼
API Gateway / Load Balancer
     │
     ├─ 1. Extract identity (user_id / API key / IP from JWT or header)
     │
     ├─ 2. Fetch rule from local rule cache
     │       └─ Cache miss → load from rules DB → update cache
     │
     ├─ 3. Check + update counter in Redis
     │       └─ Lua script (atomic read-modify-write)
     │
     ├─ 4. Decision
     │       ├─ ALLOW  → forward to backend + set X-RateLimit headers
     │       └─ REJECT → 429 Too Many Requests + Retry-After header
     │
     ▼
  Backend Service
```

### Response Headers
```
HTTP/1.1 200 OK
X-RateLimit-Limit:     100
X-RateLimit-Remaining: 43
X-RateLimit-Reset:     1700000120   ← unix timestamp of window reset

HTTP/1.1 429 Too Many Requests
Retry-After:           17           ← seconds until retry is safe
```

---

## Race Condition Handling

### Problem
Two nodes read `counter = 9`, both write `counter = 10` → actual count = 10, allowed 2 requests that should've been blocked.

### Solutions

**Option A: Redis Lua Script (Atomic)**
```lua
local key    = KEYS[1]
local limit  = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local count = redis.call('INCR', key)
if count == 1 then
  redis.call('EXPIRE', key, window)
end
if count > limit then
  return 0   -- rejected
else
  return 1   -- allowed
end
```
- Lua scripts execute atomically on Redis — no race condition
- Single round-trip to Redis

**Option B: Redis SET with NX + EXPIRE**
```
SET rl:user:window 0 NX EX 60   # init only if not exists
INCR rl:user:window              # atomic increment
```

**Option C: Async counter (eventual)**
- Update counter asynchronously in background
- May allow slightly more than limit during burst — acceptable for soft limits
- Reduces Redis round-trip from hot path

---

## Soft vs Hard Limit

| Mode | Behavior | Use When |
|------|----------|----------|
| **Hard limit** | Reject with 429 immediately | Security, cost control, SLA enforcement |
| **Soft limit** | Queue request; process when tokens available | Internal services, non-critical background jobs |
| **Throttle (slow down)** | Add delay before responding | User-facing — degrade gracefully |
| **Shadow mode** | Count but don't enforce | Testing new limits before enforcement |

---

## Distributed Rate Limiting

### Per-Node (local counter)
- Each node tracks its own counter
- Fast (no network call), but inaccurate — 10 nodes × 100 req limit = 1000 req allowed
- Use only for coarse limits with many nodes or for IP-based blocking

### Centralized Redis
- All nodes share one Redis counter
- Accurate, adds 1–2ms Redis RTT per request
- Redis cluster for HA; reads/writes to same slot (hash tag `{user_id}`)
- **Single point of failure** → use Redis Sentinel or Cluster

### Hybrid: Local + Sync
- Each node has local budget (e.g., 10% of global limit)
- Periodically sync with Redis to reclaim/return budget
- Reduces Redis calls by 10× — used by Cloudflare
- Slight overcounting possible during sync interval

---

## Failure Handling

| Redis State | Strategy | Rationale |
|-------------|----------|-----------|
| Redis down | **Fail open** (allow all) | Better UX than blocking all traffic |
| Redis slow (>5ms) | Circuit break → fail open | Don't add latency to hot path |
| Redis timeout | Fail open with logging | Alert on-call, investigate |
| Counter stale | Acceptable | ~1% overcounting is fine |

**Fail open** is standard for rate limiters — correctness in failure is secondary to availability.

---

## Where to Deploy

```
                        ┌─ API Gateway (per-client HTTP limits)
Client → CDN ──────────┤
                        └─ LB (per-IP DDoS protection)
                              │
                        App-level rate limiter (per-user business limits)
                              │
                        Service mesh (per-service circuit breaking)
```

| Layer | Limits | Tool |
|-------|--------|------|
| CDN (Cloudflare) | Per-IP DDoS, geo-block, WAF | Cloudflare WAF |
| Load Balancer | Per-IP connection rate | AWS WAF, Nginx |
| API Gateway | Per-API-key, per-user, per-endpoint | Kong, AWS API GW, Envoy |
| Application | Per-user business quota (10 AI requests/day) | Custom + Redis |
| Service Mesh | Per-service circuit breaking + retries | Istio / Envoy |

---

## Read:Write Ratio
- **Write-heavy**: every request = 1 Redis write (INCR) + 1 read
- Redis write throughput: ~100k–1M ops/sec (single node)
- At 36k peak QPS → well within single Redis node capacity

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Counter storage | Redis | Sub-ms, atomic INCR, TTL built-in |
| Algorithm | Sliding window counter | Low memory, accurate enough, no boundary burst |
| Atomicity | Lua script | Single round-trip, no race condition |
| Rule storage | DB + local cache (60s TTL) | Fast lookup, tolerate stale rules |
| Failure mode | Fail open | Availability > precision under failure |
| Identity | User ID (JWT) > API key > IP | Most granular → fairest limiting |

---

## Interview Scenarios

### "Redis goes down — should rate limiter block all traffic?"
- Fail open: allow all requests when Redis is unavailable → better UX, risk of abuse during outage
- Fail closed: block all requests → safe but causes outage for all users
- Standard choice: **fail open** for rate limiting (it's a performance layer, not security gate)
- Implementation: catch Redis connection error → log + alert → return `allowed=true` with degraded flag
- Circuit breaker: if Redis error rate >10% over 30s → switch to local in-process counter (approximate)

### "Single Redis becomes a bottleneck at 1M req/sec"
- Single Redis: ~1M ops/sec ceiling (single-threaded command execution)
- Fix: Redis Cluster — hash tag `{user_id}` ensures same user always hits same slot/node
- Or: local budget approach (Cloudflare pattern) — each app server holds 10% of user's budget; sync every 100ms; reduces Redis calls 10×
- Separate Redis cluster per rate-limit tier: critical API limits on dedicated cluster; background job limits on shared cluster

### "Rate limiter needs to enforce different limits per user tier (free: 100/hr, paid: 10k/hr)"
- Store tier in JWT claims or a fast user-profile cache (Redis hash)
- On each request: `tier = get_tier(user_id)` → `limit = rules[tier]` → check counter
- Rule refresh: cache rules in-process with 60s TTL → tolerate stale rules for 1 minute (acceptable)
- Dynamic updates: push rule changes to Redis pub/sub → all nodes invalidate local cache immediately

### "Attacker bypasses per-IP limit using 10,000 different IPs (distributed attack)"
- Per-IP limits don't stop distributed attacks; need per-user + per-behavioral pattern
- Add global rate limit: if total RPS from `/login` endpoint >10k/s → circuit break at API gateway
- Behavioral fingerprinting: limit by `(user_agent, ASN, request_pattern)` not just IP
- CAPTCHA challenge: trigger for suspicious patterns before hitting actual rate limit
- DDoS: escalate to WAF/Cloudflare — not solvable purely at rate limiter layer

### "Need rate limiting that carries over (unused quota should not roll over)"
- Fixed window: quota resets every hour regardless → no rollover (standard behavior)
- Token bucket: tokens accumulate up to bucket capacity (burst allowance) — this IS rollover
- To prevent rollover in token bucket: set `bucket_capacity = refill_rate × 1 interval` → tokens never build up beyond 1 window's worth

### "Constraint: rate limiter must add <1ms latency (current Redis adds 2ms)"
- Redis RTT = 1–2ms on same-DC; hard to get below without eliminating the round-trip
- Fix: local in-process counter + async sync → 0 network latency on hot path; sync every 500ms
- Accept slight overcounting (up to `sync_interval × rate`) — for most APIs, this is acceptable
- Ultra-low latency option: kernel-level rate limiting (eBPF/XDP) → sub-microsecond, but complex to deploy

### "Constraint: rate limit must be per organization (all users under one org share a pool)"
- Key change: `rl:{org_id}:{endpoint}:{window}` instead of `rl:{user_id}:...`
- Org-level limits require org_id from JWT or user-to-org lookup (cache org membership)
- Hierarchical limits: enforce both `per-user` AND `per-org` in same request using Lua script with two INCR calls
- Fair queuing within org: each user gets a sub-quota; org limit is the ceiling — prevents one user exhausting org pool
