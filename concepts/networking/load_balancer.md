# Load Balancer

## What It Does
- Distributes incoming traffic across multiple backend servers
- Hides topology from clients (single VIP / DNS entry)
- Health checks — removes unhealthy backends automatically
- TLS termination, connection pooling, rate limiting (L7)

---

## L4 vs L7

| | L4 (Transport Layer) | L7 (Application Layer) |
|-|---------------------|----------------------|
| **Works at** | TCP / UDP | HTTP / HTTPS / gRPC / WebSocket |
| **Sees** | IP + port only | URL, headers, cookies, body |
| **Routing basis** | IP tuple hash | Path, host, header, method |
| **TLS** | Pass-through (no termination) | Terminates TLS, re-encrypts to backend |
| **Latency** | Very low (no parsing) | Slightly higher (parse HTTP) |
| **Sticky sessions** | IP-hash only | Cookie-based, header-based |
| **Health check** | TCP connect | HTTP GET `/health` — checks app logic |
| **Use case** | Raw TCP (DB proxies, SMTP), ultra-low latency | HTTP APIs, microservices, A/B testing |
| **Examples** | AWS NLB, HAProxy TCP mode, IPVS | AWS ALB, Nginx, Envoy, HAProxy HTTP mode |

### When to Use L4
- Non-HTTP protocols (MySQL proxy, Redis proxy, SMTP)
- Need absolute minimum latency overhead
- TLS passthrough required (client cert auth end-to-end)
- Very high connection rates (millions/sec)

### When to Use L7
- HTTP/HTTPS microservices (almost always)
- Need URL-based routing (`/api` → service A, `/static` → S3)
- Header-based routing (canary, A/B, multi-tenant)
- Rate limiting, auth, request rewriting at the edge
- WebSocket upgrades, gRPC (HTTP/2)

---

## Load Balancing Algorithms

### Stateless (no per-backend state)

| Algorithm | How | Best For |
|-----------|-----|----------|
| **Round Robin** | Requests 1,2,3... distributed in order | Homogeneous servers, uniform requests |
| **Weighted Round Robin** | Each server gets weight proportional to capacity | Heterogeneous servers (different CPU/RAM) |
| **Random** | Pick backend at random | Simple, no coordination, works at scale |
| **IP Hash** | `hash(client_ip) % N` → same server per client | Weak sticky session, stateful backends |
| **URL Hash** | `hash(url) % N` → same server per URL | CDN/cache servers — same content to same node |

### Stateful (tracks backend state)

| Algorithm | How | Best For |
|-----------|-----|----------|
| **Least Connections** | Route to backend with fewest active connections | Long-lived connections (WebSocket, file upload) |
| **Weighted Least Connections** | Least connections + capacity weight | Mixed-capacity fleet with long connections |
| **Least Response Time** | Route to fastest-responding backend | Latency-sensitive, heterogeneous response times |
| **Resource-Based** | Route based on CPU/memory reported by agents | Compute-intensive workloads |

### Consistent Hashing
- Backend servers placed on a virtual ring
- Request key hashed → clockwise walk → first node
- Adding/removing a server only remaps `K/N` keys (not all)
- Used when: session affinity, cache affinity (same key to same cache node)
- Virtual nodes per server → more even distribution

```
Ring: 0 ──────── ServerA ──── ServerB ──── ServerC ──── 2^32
Request hash → clockwise → first server = owner
```

---

## Stateless vs Stateful Backends

### Stateless Backends (preferred)
- No session data stored on the app server
- Any request can go to any instance → true horizontal scale
- Session state stored externally (Redis, DB)
- **LB algorithm**: round robin / random — simple and effective

### Stateful Backends (harder to scale)
- Session data lives on a specific server
- Requires **sticky sessions** (session affinity)
- If that server dies → session lost (unless replicated)

### Sticky Sessions Implementation

| Method | How | Risk |
|--------|-----|------|
| **Cookie-based** | LB injects `Set-Cookie: SERVERID=s1`; routes by cookie | Cookie can be stripped, HTTPS-only safe |
| **IP-hash** | `hash(client_ip) % N` → always same server | Breaks with NAT (many users → same IP), CGNAT |
| **Consistent hashing** | Stable mapping via ring | Node failure remaps only adjacent keys |

**Best practice**: avoid sticky sessions — move state to Redis instead.

---

## Health Checks

| Type | How | Use |
|------|-----|-----|
| **TCP check** | Open TCP connection to port | L4 LBs; confirms port is listening |
| **HTTP check** | GET `/health` → expect 200 | L7 LBs; confirms app is alive |
| **Deep check** | `/health` checks DB + cache connectivity | Detects degraded (alive but broken) backends |
| **Passive** | Monitor error rate on real traffic | Detect degraded performance without extra probes |

- Unhealthy threshold: 2–3 consecutive failures → remove
- Healthy threshold: 2–3 consecutive successes → re-add
- **Don't couple `/health` to flaky dependencies** — cascading removal under DB slowness

---

## Rate Limiting at the Load Balancer

### Where to Rate Limit

```
Client → [CDN rate limit] → [LB/API GW rate limit] → [App rate limit] → Backend
```

- **CDN layer**: block DDoS, per-IP limits, geographic blocks
- **LB/API Gateway**: per-client token bucket, per-route limits
- **App layer**: per-user business logic limits (X requests/hour per account)

### Rate Limiting Algorithms

| Algorithm | Behavior | Memory | Use |
|-----------|----------|--------|-----|
| **Token bucket** | Refill at rate R; burst up to capacity B | O(1) per key | API gateways — allows controlled bursts |
| **Leaky bucket** | Queue requests; drain at constant rate | O(queue size) | Traffic shaping, smooth output |
| **Fixed window** | Counter per time window (1min, 1hr) | O(1) per key | Simple; edge spike at window boundary |
| **Sliding window log** | Log all request timestamps; count in window | O(requests) | Exact; memory-heavy at high QPS |
| **Sliding window counter** | Weighted interpolation of two fixed windows | O(1) per key | Approximate; production standard |

### Sliding Window Counter (most common)
```
current_window_count = prev_window_count × (1 - elapsed/window) + current_count

Example: window=60s, elapsed=45s, prev=80, curr=30
→ rate = 80 × (1 - 45/60) + 30 = 80×0.25 + 30 = 50 requests in window
```

### Distributed Rate Limiting
- Single server: in-process counter (fastest)
- Multi-node: Redis atomic counter (`INCR` + `EXPIRE`) or Redis Lua script for atomicity
- Tradeoff: Redis adds ~1–2ms per check; acceptable for most API GWs

```lua
-- Redis atomic rate limit check (Lua)
local count = redis.call('INCR', key)
if count == 1 then redis.call('EXPIRE', key, window_seconds) end
return count
```

---

## Connection Handling

### Connection Pooling (L7)
- LB maintains persistent connections to backends (keep-alive)
- Client opens new connection → LB reuses existing backend connection
- Avoids TCP handshake overhead per request
- Critical for DB proxies (PgBouncer, ProxySQL)

### SSL/TLS Termination
```
Client ──[TLS]──► LB ──[plain HTTP or re-encrypted]──► Backend

Option A: Terminate at LB → plain HTTP to backend (faster, less secure internally)
Option B: Terminate at LB → re-encrypt to backend (mTLS) — zero-trust
Option C: TLS passthrough → backend handles TLS (L4 only, client cert auth)
```

### HTTP/2 and gRPC
- L7 LBs must support HTTP/2 to load balance gRPC (stream-level, not connection-level)
- HTTP/2 multiplexes many requests on one connection → naive L4 sends all to one backend
- Envoy/Nginx with HTTP/2 do proper per-request (stream) load balancing

---

## Global Load Balancing (GeoDNS / Anycast)

### DNS-Based (GeoDNS)
- DNS resolver returns different IPs based on client location
- Route US users → US region, EU users → EU region
- TTL = 30–60s; failover is slow (TTL must expire)
- Used by: AWS Route53 latency routing, Cloudflare, Akamai

### Anycast
- Same IP announced from multiple locations via BGP
- Network routing sends client to nearest PoP automatically
- Instant failover (BGP reconverges in ~seconds)
- Used by: Cloudflare (1.1.1.1), Google (8.8.8.8), CDN PoPs

### Active-Passive vs Active-Active
| Mode | Write | Read | Failover |
|------|-------|------|---------|
| Active-Passive | Primary only | Primary only | Promote passive (~30–60s) |
| Active-Active | Both regions | Both regions | Instant (no failover needed) |
| Active-Active w/ conflict | Both | Both | Requires CRDT / last-write-wins |

---

## Service Mesh (L7 in Sidecar)

- LB logic moves into a sidecar proxy (Envoy) next to every service instance
- No centralized LB — each pod has its own proxy
- Features: mTLS between services, retries, circuit breaking, distributed tracing, traffic splitting

```
Service A → Envoy sidecar → [mTLS] → Envoy sidecar → Service B
```

Used by: Istio, Linkerd, AWS App Mesh, Consul Connect

---

## Key Numbers

| Metric | Typical Value |
|--------|--------------|
| L4 LB max connections | 1–10M concurrent (NLB) |
| L7 LB max RPS | 100k–1M RPS (ALB, Nginx) |
| Health check interval | 5–30s |
| Unhealthy threshold | 2–3 failures |
| Connection timeout | 30–60s (idle) |
| Rate limit Redis check overhead | 1–2ms |
| DNS TTL for failover | 30–60s |
| Anycast BGP failover | ~seconds |

---

## Summary: Pick the Right LB

| Need | Choice |
|------|--------|
| HTTP microservices, TLS, routing | L7 (ALB / Nginx / Envoy) |
| Raw TCP, DB proxy, ultra-low latency | L4 (NLB / HAProxy TCP) |
| Stateful app, can't move to Redis | Sticky session via cookie |
| Cache/session locality | Consistent hashing |
| Heterogeneous fleet | Weighted round robin |
| Long-lived connections (WS, upload) | Least connections |
| Global traffic routing | GeoDNS + Anycast |
| Service-to-service (k8s) | Service mesh (Envoy sidecar) |
| Rate limiting distributed API | Sliding window counter + Redis |
