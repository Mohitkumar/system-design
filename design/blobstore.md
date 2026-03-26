# Blob Store — System Design

## What Is It
Object storage for arbitrary binary data (images, video, backups, logs, model weights).
Flat namespace: **account → container/bucket → blob/object**.

**Examples**: Amazon S3, Azure Blob Storage, Google Cloud Storage, Cloudflare R2, MinIO

---

## Functional Requirements
- PUT blob (upload, multipart for large files)
- GET blob (full or byte-range)
- DELETE blob (soft + hard delete)
- LIST blobs in container (paginated, prefix filter)
- CREATE / DELETE / LIST containers (buckets)
- Access control (public, private, signed URLs, ACL/IAM policies)

## Non-Functional Requirements
- **Durability**: 11 nines (99.999999999%) — data never lost
- **Availability**: 99.99% read, 99.9% write
- **Throughput**: high (video streaming, bulk upload/download)
- **Latency**: eventual for global replication; first-byte < 200ms in-region
- **Scale**: exabytes of data, billions of objects
- Strongly consistent within a region; eventually consistent cross-region

---

## Capacity Estimation

```
DAU:             5M users
Avg upload/day:  2 blobs × 50 MB video + 10 KB thumbnail = ~100 MB/user/day
Write QPS:       5M × 2 / 86,400 ≈ 115 writes/sec (blobs)
Read QPS:        read:write ≈ 10:1 → ~1,150 reads/sec (80:20 → peak ~3.5k)

Daily ingress:   5M × 100 MB = 500 TB/day
5yr storage:     500 TB × 1,825 days × 3 (replicas) ≈ 2.7 EB

Bandwidth:
  Write: 115 writes/s × 50 MB = ~5.7 GB/s ingress
  Read:  1,150 reads/s × 50 MB = ~57 GB/s egress (before CDN)
  After CDN offload (80%): ~11 GB/s origin egress
```

---

## Architecture

```
Client
  │
  ├─ Small blob (<100 MB) ──► API Gateway ──► Front-End Servers
  │                                                   │
  └─ Large blob (multipart) ──► Multipart initiate ──┤
                                                      │
                                ┌─────────────────────┤
                                │                     │
                          Metadata Store        Chunk Servers
                         (partition map,         (actual bytes)
                          ACL, tags, state)
                                │                     │
                          Partition Layer        Replication
                         (route by acct+cont+blob)  (3× in-region
                                                  + async cross-region)
```

### Components

| Component | Role |
|-----------|------|
| **API Gateway** | Auth, TLS termination, request routing, signed URL validation |
| **Front-End Servers** | Stateless; parse request, check ACL, coordinate multipart, stream data |
| **Metadata Store** | Blob metadata: name, size, checksum, ACL, tags, state, chunk locations |
| **Chunk Servers (Data Nodes)** | Store fixed-size chunks of blob data on disk |
| **Partition Manager** | Maps `(account, container, blob)` → responsible chunk servers |
| **Garbage Collector** | Reclaims space for DELETED / orphaned chunks |
| **Replication Manager** | Maintains replication factor; heals under-replicated chunks |
| **CDN** | Caches frequently accessed public blobs at edge PoPs |

---

## Data Model

### Blob Key (Partition Key)
```
/{account_id}/{container_id}/{blob_name}

Lexicographically sorted → range scans for LIST prefix
Partition by: hash(account_id + container_id) → avoid hot accounts
```

### Metadata Record
```
blob_id        : UUID (internal)
account_id     : string
container_id   : string
blob_name      : string
size_bytes     : int64
content_type   : string
checksum_md5   : bytes (integrity)
checksum_sha256: bytes (integrity)
state          : ACTIVE | DELETED | PENDING
chunk_list     : [{chunk_id, offset, server_id}]
acl            : {owner, public_read, signed_url_expiry}
tags           : map<string,string>       ← user-defined key-value tags
created_at     : timestamp
last_modified  : timestamp
version_id     : UUID (if versioning enabled)
etag           : MD5 of content (for conditional requests)
```

### Chunk Storage Layout
```
Fixed chunk size: 64 MB (configurable)

Blob 500 MB → 8 chunks × 64 MB
Each chunk: {chunk_id, blob_id, offset, data, checksum}

Physical file on data node:
  /data/vol-01/{chunk_id}.bin     ← raw bytes, append-only
  /data/vol-01/{chunk_id}.meta    ← offset index for byte-range reads
```

---

## Write Path

### Small Blob (< 100 MB — single PUT)
```
1. Client → PUT /bucket/blob
2. Front-end: authenticate, check ACL, generate blob_id
3. Front-end → Partition Manager: which chunk servers?
4. Front-end streams data → primary chunk server
5. Primary: write to local disk, replicate to 2 secondaries (sync or async)
6. On quorum ACK: front-end → Metadata Store: commit metadata (state=ACTIVE)
7. Return ETag (MD5) + 200 OK to client
```

### Large Blob (multipart upload)
```
1. POST /bucket/blob?uploads → returns UploadId
2. Client uploads N parts in parallel:
   PUT /bucket/blob?partNumber=1&uploadId=X  → ETag per part
   PUT /bucket/blob?partNumber=2&uploadId=X
   ...
3. POST /bucket/blob?uploadId=X  (complete multipart)
   Body: [{ partNumber, ETag }]
4. Server assembles chunk list, writes final metadata → ACTIVE
5. Background: stitch parts into contiguous layout (or keep as part list)
```

**Why multipart?**
- Parallelism: 10 × 100 MB parts = 10× upload speed
- Resumability: failed parts re-uploaded independently
- S3 minimum part: 5 MB; max parts: 10,000 → max object: 5 TB

---

## Read Path

### Full Blob
```
1. Client → GET /bucket/blob
2. Front-end: auth, fetch metadata → get chunk list + server locations
3. Front-end: stream chunks sequentially from chunk servers
4. Return bytes to client (or redirect to CDN if cached)
```

### Byte-Range Read (video seek / partial download)
```
GET /bucket/video.mp4
Range: bytes=10000000-20000000

1. Metadata lookup → chunk list
2. Calculate which chunks cover byte range
   chunk_idx  = offset / chunk_size
   chunk_off  = offset % chunk_size
3. Fetch only relevant chunks (1–2 chunks for typical seek)
4. Slice and return requested bytes
```
Byte-range reads are critical for video streaming — avoid re-downloading entire file.

---

## Replication Strategy

### In-Region (Synchronous — 3× default)
```
Write → Primary chunk server
      → Replication pipeline: Primary → Secondary1 → Secondary2
      → Quorum ACK (2 of 3) before client response
```

- Rack-aware: replicas on different racks (survive rack failure)
- AZ-aware: replicas in different availability zones (survive AZ failure)

### Cross-Region (Asynchronous)
```
Primary region commits → async replication queue
→ Secondary region chunk servers receive and apply
→ Lag: seconds to minutes (eventually consistent cross-region)
```

- Cross-region replication lag must be tolerated by clients
- Strong consistency: always read from primary region
- **S3 approach**: same-region reads = strong consistency (since 2020); cross-region = eventual

### Erasure Coding (for cold storage)
```
Instead of 3× replication (200% overhead):
Reed-Solomon (6, 3): split object into 6 data shards + 3 parity shards
Store 9 shards across 9 different nodes
Survive any 3 node failures → reconstruct from any 6 shards
Storage overhead: 9/6 = 1.5× (vs 3× for replication)
```

| Scheme | Overhead | Durability | Read Latency | Use |
|--------|----------|------------|-------------|-----|
| 3× replication | 200% | Very high | Low (any replica) | Hot data, low latency |
| Erasure (6,3) | 50% | Equivalent | Higher (reconstruct if degraded) | Cold/warm data, cost |
| Erasure (10,4) | 40% | Very high | Higher | Archive (Glacier, GCS Coldline) |

S3 Standard uses erasure coding internally; S3 Glacier uses deeper erasure for cost.

---

## Metadata Store Design

### Requirements
- Strong consistency within region
- High read throughput (every GET/PUT reads metadata)
- Partition by `(account_id, container_id)`

### Options
| Store | Why |
|-------|-----|
| **DynamoDB / Cassandra** | Horizontally scalable, partition by account+container |
| **PostgreSQL + sharding** | ACID, range queries for LIST, complex ACL joins |
| **Azure Cosmos DB** | Azure Blob uses this internally — multi-model, geo-distribution |

### Partition Key Choice
```
Good:   hash(account_id + container_id)  → even distribution
Bad:    sequential blob_id               → hot partition on latest writes
Bad:    account_id only                  → hot account (large customer) overwhelms one shard
```

---

## Blob Index (Tag-Based Search)

```
User tags: { "project": "video-2024", "status": "processed" }

Index table:
  tag_key + tag_value + account_id → [blob_id, blob_id, ...]

Query: LIST blobs WHERE tag:project = "video-2024"
→ index lookup → blob_ids → fetch metadata
```

**S3 equivalent**: S3 Object Tagging + S3 Batch Operations
**Azure equivalent**: Blob Index Tags (separate from metadata, indexed by storage)

---

## Consistency Models

| Operation | S3 (AWS) | Azure Blob | GCS |
|-----------|----------|-----------|-----|
| Single-region PUT → GET | Strong (since Dec 2020) | Strong | Strong |
| DELETE → LIST | Strong | Strong | Strong |
| Cross-region replication | Eventual | Eventual | Eventual |
| List after PUT | Strong | Strong | Strong |

**S3 before 2020**: eventually consistent (PUT then immediate GET could miss). Now strong by default — no more cache tricks needed.

---

## Access Control

### Patterns
| Pattern | How | Use |
|---------|-----|-----|
| **Private** | Require auth (IAM role, access key) on every request | Internal data |
| **Public read** | Bucket policy allows anonymous GET | Public website assets |
| **Pre-signed URL** | Time-limited signed URL (`?X-Amz-Expires=3600&X-Amz-Signature=...`) | Share private files without credentials |
| **IAM Policy** | JSON policy attached to user/role: allow/deny per action/resource | Fine-grained internal access |
| **Bucket ACL** | Legacy (S3); object-level allow list | Avoid — use IAM policies instead |
| **SAS Token (Azure)** | Signed URL with permissions bitmask + expiry | Azure equivalent of pre-signed URL |

### Pre-signed URL Flow
```
1. Server generates: sign(bucket + key + expiry + permissions, secret_key)
2. Server returns URL to client
3. Client uploads/downloads directly to blob store (bypasses server)
→ Server never touches the bytes → massive throughput win
```

---

## Garbage Collection

### Why Needed
- DELETE marks blob as `DELETED` in metadata — bytes still on disk
- Multipart upload abandoned — parts orphaned
- Overwritten object — old version chunks dangling

### GC Process
```
1. Periodic GC job scans metadata for state=DELETED + grace period expired
2. Resolves all chunk IDs referenced by that blob
3. Cross-checks chunk reference counts (no other blob points to same chunk)
4. Marks chunks for deletion on chunk servers
5. Chunk servers free disk space during off-peak compaction
6. Remove metadata record
```

- Grace period (e.g., 24h–7 days) allows accidental delete recovery
- Soft delete → `DELETED` state (recoverable)
- Hard delete → GC removes bytes (irrecoverable)

---

## Caching Strategy

| Cache Layer | What | TTL | Tool |
|-------------|------|-----|------|
| Client / browser | GET response + ETag | Via `Cache-Control` header | Browser |
| CDN | Popular public blobs | Minutes to days | CloudFront, Fastly |
| Front-end server | Partition map (account→chunk servers) | 60s | In-process |
| Metadata cache | Blob metadata (location, size, ACL) | 30s | Redis |
| Chunk server | Hot chunk data | LRU, fit in RAM | In-process |

**Cache invalidation**: on write/delete, invalidate metadata cache key `(account, container, blob)`.
ETag + `If-None-Match` header for conditional GETs — return 304 if unchanged.

---

## S3 vs Azure Blob vs GCS — Design Differences

| Aspect | Amazon S3 | Azure Blob Storage | Google Cloud Storage |
|--------|-----------|-------------------|---------------------|
| **Namespace** | Globally unique bucket names | Account → Container → Blob | Project → Bucket → Object |
| **Consistency** | Strong (single-region) | Strong | Strong |
| **Storage tiers** | Standard, IA, Glacier Instant, Glacier, Deep Archive | Hot, Cool, Cold, Archive | Standard, Nearline, Coldline, Archive |
| **Max object size** | 5 TB (multipart) | 4.75 TB (block blob) | 5 TB |
| **Multipart** | 5 MB min part, 10k max parts | Block blobs: 50k blocks × 4000 MB | Composite objects (32 max) |
| **Block model** | Monolithic parts | Block list — stage blocks, commit list | Compose operation |
| **Blob types** | Object (single type) | Block blob / Append blob / Page blob | Object (single type) |
| **Append blob** | No native append | Append blob type — log/audit use | No native append |
| **Page blob** | No (use EBS) | Page blob — 512-byte pages, used for VHD/disk | No |
| **Versioning** | Optional, per-bucket | Optional, per-container | Optional, per-bucket |
| **Replication** | Cross-Region Replication (CRR) | GRS/GZRS — geo-redundant | Multi-region buckets |
| **Event notifications** | S3 Event Notifications → SNS/SQS/Lambda | Event Grid → Azure Functions | Pub/Sub notifications |
| **Lifecycle rules** | Transition to cheaper tier / expire | Lifecycle management policies | Object lifecycle management |
| **CDN integration** | CloudFront | Azure CDN / Front Door | Cloud CDN |
| **Direct upload** | Pre-signed URL | SAS Token | Signed URL |
| **Pricing model** | Per GB-month + per request + egress | Per GB-month + per operation + egress | Per GB-month + per op + egress (free between GCP services) |

### Azure Blob Types (unique to Azure)
```
Block Blob:   Default. Data stored as blocks; commit a block list.
              Best for: images, video, documents, general objects.

Append Blob:  Can only append blocks; no modify/overwrite.
              Best for: log files, audit trails, telemetry streaming.
              Each append is atomic; no read-modify-write race.

Page Blob:    Random read/write on 512-byte pages.
              Best for: Azure VM disks (VHD), database files.
              Effectively a virtual disk.
```

---

## Lifecycle Management (Tiering)

```
New data      → Hot/Standard tier     (low latency, high cost)
After 30 days → Cool/Infrequent Access (slightly higher retrieval latency)
After 90 days → Cold tier             (hours retrieval, cheap storage)
After 1 year  → Archive/Glacier       (retrieval: minutes–hours, very cheap)
```

**S3 Lifecycle Rule example**:
```json
{
  "Rules": [{
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER_IR"  },
      { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "Expiration": { "Days": 2555 }
  }]
}
```

Cost comparison (approx, S3):
| Tier | Storage/GB/month | Min storage duration | Retrieval |
|------|-----------------|---------------------|-----------|
| Standard | $0.023 | None | Free |
| Standard-IA | $0.0125 | 30 days | $0.01/GB |
| Glacier Instant | $0.004 | 90 days | $0.03/GB |
| Deep Archive | $0.00099 | 180 days | $0.02/GB |

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Chunk size | 64 MB | Balance between metadata overhead and I/O efficiency |
| Replication | 3× sync in-region, async cross-region | Durability + availability without blocking on geo latency |
| Cold data | Erasure coding (6,3) | 50% overhead vs 200% for replication |
| Metadata store | Cassandra / DynamoDB (partitioned by account+container) | Scale to billions of objects |
| Delete | Soft delete + GC | Accidental delete recovery + atomic reference cleanup |
| Large upload | Multipart | Parallelism + resumability |
| Client upload | Pre-signed URL (direct to storage) | Server never handles bytes → massive throughput |
| Consistency | Strong single-region, eventual cross-region | Practical balance of latency vs correctness |
