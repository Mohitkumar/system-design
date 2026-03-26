# Capacity Planning

## Power of 2

| Power | Exact Value | Approx | Unit |
|-------|-------------|--------|------|
| 7 | 128 | | |
| 8 | 256 | | |
| 10 | 1,024 | 1 thousand | 1 KB |
| 16 | 65,536 | | 64 KB |
| 20 | 1,048,576 | 1 million | 1 MB |
| 30 | 1,073,741,824 | 1 billion | 1 GB |
| 32 | 4,294,967,296 | | 4 GB |
| 40 | 1,099,511,627,776 | 1 trillion | 1 TB |

---

## Latency Numbers (Every Engineer Should Know)

| Operation | Nanoseconds | Microseconds | Milliseconds | Notes |
|-----------|-------------|-------------|-------------|-------|
| L1 cache reference | 0.5 ns | | | |
| Branch mispredict | 5 ns | | | |
| L2 cache reference | 7 ns | | | 14× L1 cache |
| Mutex lock/unlock | 25 ns | | | |
| Main memory reference | 100 ns | | | 20× L2, 200× L1 |
| Compress 1KB (Snappy) | 10,000 ns | 10 µs | | |
| Send 1KB over 1Gbps | 10,000 ns | 10 µs | | |
| Read 4KB randomly from SSD | 150,000 ns | 150 µs | | ~1 GB/s SSD |
| Read 1MB sequentially from RAM | 250,000 ns | 250 µs | | |
| Round trip within same DC | 500,000 ns | 500 µs | | |
| Read 1MB sequentially from SSD | 1,000,000 ns | 1,000 µs | 1 ms | 4× memory |
| HDD seek | 10,000,000 ns | 10,000 µs | 10 ms | 20× DC round trip |
| Read 1MB sequentially over 1Gbps | 10,000,000 ns | 10,000 µs | 10 ms | 10× SSD |
| Read 1MB sequentially from HDD | 30,000,000 ns | 30,000 µs | 30 ms | 30× SSD |
| Packet CA → Netherlands → CA | 150,000,000 ns | 150,000 µs | 150 ms | |

```
1 ns = 10^-9 s
1 µs = 10^-6 s = 1,000 ns
1 ms = 10^-3 s = 1,000 µs = 1,000,000 ns
```

### Key Takeaways
- Memory is 20× faster than network within DC, 200× faster than inter-continental
- SSD random read ~1000× slower than L1 cache
- HDD seek ~20× slower than SSD random read
- Avoid cross-continent calls in the hot path — 150ms is a full user-perceptible delay

---

## Capacity Estimation Framework

### Step 1 — Define Scale
| Metric | Question |
|--------|----------|
| DAU | How many daily active users? |
| Actions/user/day | How many reads/writes per user? |
| Data size per action | How large is each request/record? |
| Retention | How long is data stored? |

---

### Step 2 — Throughput (QPS)

**Method A: DAU-based**
```
avg QPS = (DAU × actions_per_day) / 86,400
```

**Method B: Direct (IoT, sensors)**
```
QPS = total_devices / reporting_interval_seconds
```

**Common DAU → QPS estimates (500M DAU example):**
| Action | Formula | QPS |
|--------|---------|-----|
| Homepage view | 500M × 10 / 86,400 | ~58k |
| Profile visit | 500M × 2 / 86,400 | ~12k |
| Content creation (10% create) | 500M × 0.1 × 1 / 86,400 | ~580 |

---

### Step 3 — Peak QPS (80:20 Rule)

> 80% of daily traffic occurs in 20% of the day (~5 hours)

```
peak QPS = (daily_requests × 0.8) / (peak_hours × 3600)
```

**Example**: 4B daily requests → `4B × 0.8 / (4 × 3600)` ≈ **222k peak QPS**

**Rule of thumb**: peak QPS ≈ **2–3× average QPS**

---

### Step 4 — Server Instances

```
workers_per_instance = 1 / response_time_seconds
QPS_per_instance     = workers × parallelism_factor
instances_needed     = peak_QPS / QPS_per_instance
```

**Example** (200ms response time, 32 workers):
```
workers_per_instance = 1 / 0.2 = 5 req/s per worker
QPS per instance     = 32 × 5  = 160 QPS
instances needed     = 200k / 160 ≈ 1,250 instances
```

Add **20–30% headroom** for spikes.

---

### Step 5 — Storage

```
daily_storage  = QPS × seconds_per_day × record_size
               = QPS × 86,400 × bytes_per_record

total_storage  = daily_storage × retention_days × replication_factor
```

**Example** (write QPS=1k, 1KB/record, 5yr retention, 3× replication):
```
daily  = 1,000 × 86,400 × 1KB  ≈ 86 GB/day
total  = 86 GB × 1,825 days × 3 ≈ 470 TB
```

---

### Step 6 — Bandwidth

```
ingress = request_size × write_QPS
egress  = response_size × read_QPS
```

**Egress optimizations** (reduces naive bandwidth by ~67%):
- Serve thumbnails (30KB) instead of full images (300KB); load full on demand
- Text excerpts, not full content
- No video autoplay; only load on explicit action
- 80% of users don't scroll past 5 posts

---

## Common Size Reference

| Object | Approx Size |
|--------|-------------|
| Integer / float | 4 bytes |
| UUID | 16 bytes |
| Timestamp | 8 bytes |
| Short string (50 chars) | ~50 bytes |
| Tweet / short post (250 chars) | ~1 KB |
| IoT sensor payload (JSON + HTTP) | ~0.5 KB |
| Web page HTML | ~50 KB |
| Compressed image (thumbnail) | ~30 KB |
| Compressed image (full) | ~300 KB |
| Video (1 min, compressed) | ~1 MB |

---

## Quick Estimation Cheatsheet

| Metric | Rule of Thumb |
|--------|--------------|
| Peak QPS | 2–3× average QPS |
| Peak storage growth | 2× baseline estimate |
| Read:Write ratio (social) | 100:1 |
| Read:Write ratio (messaging) | 1:1 |
| Cache hit rate target | ≥ 99% for hot data |
| Replication factor | 3× (standard) |
| 1 server (commodity) | ~1k–10k QPS depending on work |
| 1 server RAM | 64–256 GB |
| 1 server disk (SSD) | 1–10 TB |
| 1 server NIC | 1–10 Gbps |

---

## Estimation Template

```
1. Scale
   DAU:             ___M
   Avg QPS:         (DAU × actions) / 86,400 = ___k
   Peak QPS:        avg × 2–3 = ___k

2. Storage
   Record size:     ___ bytes/KB
   Daily write vol: write_QPS × 86,400 × size = ___ GB/day
   5yr total:       daily × 1,825 × 3 (replica) = ___ TB

3. Bandwidth
   Ingress:         write_QPS × request_size = ___ MB/s
   Egress:          read_QPS × response_size = ___ GB/s
   After optim.:    egress × 0.33 = ___ GB/s

4. Servers
   instances:       peak_QPS / QPS_per_server = ___
   + 30% headroom: ___ instances
```
