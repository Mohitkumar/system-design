# Blob Store — System Design

## What Is It
Object storage for arbitrary binary data (images, video, backups, logs, model weights).
Flat namespace: **account → bucket → object** (prefixes simulate subfolders).

**Examples**: Amazon S3, Azure Blob Storage, Google Cloud Storage, Cloudflare R2, MinIO

---

## Functional Requirements
- PUT object (upload, multipart for large files)
- GET object (full or byte-range)
- DELETE object (soft + hard delete)
- LIST objects in bucket (paginated, prefix filter)
- CREATE / DELETE / LIST buckets
- Access control (public, private, pre-signed URLs, ACL/IAM policies)

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
Avg upload/day:  2 objects × 50 MB video + 10 KB thumbnail = ~100 MB/user/day
Write QPS:       5M × 2 / 86,400 ≈ 115 writes/sec (objects)
Read QPS:        read:write ≈ 10:1 → ~1,150 reads/sec (80:20 → peak ~3.5k)

Daily ingress:   5M × 100 MB = 500 TB/day
5yr storage:     500 TB × 1,825 days × 3 (replicas) ≈ 2.7 EB

Bandwidth:
  Write: 115 writes/s × 50 MB = ~5.7 GB/s ingress
  Read:  1,150 reads/s × 50 MB = ~57 GB/s egress (before CDN)
  After CDN offload (80%): ~11 GB/s origin egress
```

---

## Namespace Model

```
account
  └── bucket (globally unique name, e.g. "my-videos-prod")
        └── object key  (e.g. "uploads/2024/jan/video001.mp4")
                │
                └── prefix "uploads/2024/jan/" simulates a subfolder
                    (no actual directory — just a key prefix convention)

LIST bucket prefix="uploads/2024/jan/" delimiter="/"
  → returns: CommonPrefixes = ["uploads/2024/jan/subfolder/"]
             Objects        = ["uploads/2024/jan/video001.mp4", ...]
```

- Objects are **flat** — no real directory tree
- Subfolders are a UI/API convention: prefix + delimiter
- Bucket names are globally unique (DNS-compatible)
- Object key max: 1,024 bytes

---

## Architecture

```
Client
  │
  ├─ Small object (<100 MB) ──► API Gateway ──► Front-End Servers
  │                                                   │
  └─ Large object (multipart) ─► Multipart initiate ─┤
                                                      │
                                ┌─────────────────────┤
                                │                     │
                          Metadata Store        Chunk Servers
                         (partition map,         (actual bytes)
                          ACL, tags, state)
                                │                     │
                          Partition Layer        Replication
                         (route by acct+bucket+key)  (3× in-region
                                                  + async cross-region)
```

### Components

| Component | Role |
|-----------|------|
| **API Gateway** | Auth, TLS termination, request routing, pre-signed URL validation |
| **Front-End Servers** | Stateless; parse request, check ACL, coordinate multipart, stream data |
| **Metadata Store** | Object metadata: key, size, checksum, ACL, tags, state, chunk locations |
| **Chunk Servers (Data Nodes)** | Store fixed-size chunks of object data on disk |
| **Partition Manager** | Maps `(account, bucket, object_key)` → responsible chunk servers |
| **Garbage Collector** | Reclaims space for DELETED / orphaned chunks |
| **Replication Manager** | Maintains replication factor; heals under-replicated chunks |
| **CDN** | Caches frequently accessed public objects at edge PoPs |

---

## Data Model

### Object Key (Partition Key)
```
/{account_id}/{bucket_name}/{object_key}

Examples:
  /acct-123/my-videos-prod/uploads/2024/jan/video001.mp4
  /acct-123/my-videos-prod/thumbnails/2024/jan/video001.jpg

Lexicographically sorted → range scans for LIST prefix queries
Partition by: hash(account_id + bucket_name) → avoid hot accounts
Internal: hash prefix added transparently to distribute sequential keys
```

### Metadata Record
```
object_id      : UUID (internal)
account_id     : string
bucket_name    : string
object_key     : string           ← full key including prefix/subfolder path
size_bytes     : int64
content_type   : string
checksum_md5   : bytes            ← integrity + ETag
checksum_sha256: bytes            ← integrity
state          : ACTIVE | DELETED | PENDING
chunk_list     : [{chunk_id, offset, server_id}]
acl            : {owner, public_read, pre_signed_expiry}
tags           : map<string,string>    ← user-defined key-value tags
user_metadata  : map<string,string>    ← x-amz-meta-* headers
created_at     : timestamp
last_modified  : timestamp
version_id     : UUID (if bucket versioning enabled)
etag           : MD5 of content (for conditional GET If-None-Match)
storage_class  : STANDARD | IA | GLACIER | DEEP_ARCHIVE
```

### Chunk Storage Layout
```
Fixed chunk size: 64 MB (configurable)

Object 500 MB → 8 chunks × 64 MB
Each chunk: {chunk_id, object_id, offset, data, checksum}

Physical file on data node:
  /data/vol-01/{chunk_id}.bin     ← raw bytes, append-only
  /data/vol-01/{chunk_id}.meta    ← offset index for byte-range reads
```

---

## Write Path

### Small Object (< 100 MB — single PUT)
```
1. Client → PUT /bucket-name/uploads/2024/jan/video001.mp4
2. Front-end: authenticate (SigV4 / API key), check bucket ACL + IAM policy
3. Front-end → Partition Manager: which chunk servers for this bucket+key?
4. Front-end streams data → primary chunk server
5. Primary: write to local disk, replicate to 2 secondaries (sync, quorum ACK)
6. On quorum ACK: front-end → Metadata Store: commit metadata (state=ACTIVE)
7. Return ETag (MD5) + 200 OK to client
```

### Large Object (multipart upload)
```
1. POST /bucket-name/object-key?uploads → returns UploadId

2. Client uploads N parts in parallel:
   PUT /bucket-name/object-key?partNumber=1&uploadId=X  → ETag per part
   PUT /bucket-name/object-key?partNumber=2&uploadId=X
   ...

3. POST /bucket-name/object-key?uploadId=X  (complete multipart)
   Body: [{ partNumber, ETag }]

4. Server assembles chunk list, writes final metadata → ACTIVE
5. Background: stitch parts into contiguous layout (or keep as part list)
```

**Why multipart?**
- Parallelism: 10 × 100 MB parts = 10× upload speed
- Resumability: failed parts re-uploaded independently
- Minimum part: 5 MB; max parts: 10,000 → max object: 5 TB

---

## Read Path

### Full Object
```
1. Client → GET /bucket-name/uploads/2024/jan/video001.mp4
2. Front-end: auth, fetch metadata by (account, bucket, key) → chunk list + server locations
3. Front-end: stream chunks sequentially from chunk servers
4. Return bytes to client (or redirect to CDN if cached)
```

### Byte-Range Read (video seek / partial download)
```
GET /bucket-name/uploads/2024/jan/video001.mp4
Range: bytes=10000000-20000000

1. Metadata lookup → chunk list
2. Calculate which chunks cover byte range:
   chunk_idx = offset / chunk_size
   chunk_off = offset % chunk_size
3. Fetch only relevant chunks (1–2 chunks for typical seek)
4. Slice and return requested bytes → 206 Partial Content
```
Byte-range reads are critical for video streaming — avoid re-downloading entire object.

---

## Subfolder / Prefix Operations

### LIST with Prefix (simulated folder listing)
```
LIST /bucket-name?prefix=uploads/2024/jan/&delimiter=/

Returns:
  CommonPrefixes: ["uploads/2024/jan/subfolder-a/", "uploads/2024/jan/subfolder-b/"]
  Objects:        ["uploads/2024/jan/video001.mp4", "uploads/2024/jan/video002.mp4"]

CommonPrefixes = "virtual subdirectories" (keys that contain another / after the prefix)
Objects        = actual objects directly under this prefix

Pagination: NextContinuationToken for next page (cursor-based, not offset)
```

### Bulk Delete by Prefix
```
No native "delete folder" — must LIST prefix then DELETE each object
S3 Batch Operations: create inventory → run delete job across billions of objects
```

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

### Erasure Coding (for cold storage)
```
Reed-Solomon (6, 3): split object into 6 data shards + 3 parity shards
Store 9 shards across 9 different nodes (spread across 3+ AZs)
Survive any 3 node failures → reconstruct from any 6 shards
Storage overhead: 9/6 = 1.5× (vs 3× for replication)
```

| Scheme | Overhead | Durability | Read Latency | Use |
|--------|----------|------------|-------------|-----|
| 3× replication | 200% | Very high | Low (any replica) | Hot data, low latency |
| Erasure (6,3) | 50% | Equivalent | Higher (reconstruct if degraded) | Cold/warm data, cost |
| Erasure (10,4) | 40% | Very high | Higher | Archive (Glacier, GCS Coldline) |

---

## Metadata Store Design

### Partition Key Choice
```
Good:   hash(account_id + bucket_name)        → even distribution
Bad:    sequential object_id                  → hot partition on latest writes
Bad:    account_id only                       → hot account (large customer) overwhelms one shard
Bad:    bucket_name only                      → hot bucket overwhelms one shard
```

### Options
| Store | Why |
|-------|-----|
| **DynamoDB / Cassandra** | Horizontally scalable, partition by account+bucket |
| **PostgreSQL + sharding** | ACID, range queries for LIST prefix, complex ACL joins |

---

## Object Index (Tag-Based Search)

```
User tags: { "project": "video-2024", "status": "processed" }

Index table:
  tag_key + tag_value + account_id → [object_key, object_key, ...]

Query: LIST objects WHERE tag:project = "video-2024"
→ index lookup → object_keys → fetch metadata
```

S3 equivalent: S3 Object Tagging + S3 Batch Operations for tag-based filtering.

---

## Consistency Models

| Operation | S3 (AWS) | Azure Blob | GCS |
|-----------|----------|-----------|-----|
| Single-region PUT → GET | Strong (since Dec 2020) | Strong | Strong |
| DELETE → LIST | Strong | Strong | Strong |
| Cross-region replication | Eventual | Eventual | Eventual |
| LIST after PUT | Strong | Strong | Strong |

---

## Access Control

| Pattern | How | Use |
|---------|-----|-----|
| **Private** | Require auth (IAM role, access key) on every request | Internal data |
| **Public read** | Bucket policy allows anonymous GET | Public website assets |
| **Pre-signed URL** | Time-limited signed URL (`?Expires=3600&Signature=...`) | Share private objects without credentials |
| **IAM Policy** | JSON policy on user/role: allow/deny per action/bucket/prefix | Fine-grained internal access |
| **Bucket Policy** | JSON policy on bucket: cross-account, public access rules | Cross-account access, CDN origins |

### Pre-signed URL Flow
```
1. Server: sign(bucket + object_key + expiry + method, secret_key) → URL
2. Server returns URL to client
3. Client PUT/GET directly to storage (bypasses your server)
→ Server never touches the bytes → bandwidth goes directly client ↔ storage
```

---

## Garbage Collection

### Why Needed
- DELETE marks object as `DELETED` in metadata — bytes still on disk
- Multipart upload abandoned — parts orphaned on chunk servers
- Overwritten object — old version chunks dangling

### GC Process
```
1. Periodic GC job scans metadata for state=DELETED + grace period expired
2. Resolves all chunk IDs referenced by that object_key
3. Cross-checks chunk reference counts (no other object/version points to same chunk)
4. Marks chunks for deletion on chunk servers
5. Chunk servers free disk space during off-peak compaction
6. Remove metadata record
```

- Grace period (24h–7 days) allows accidental delete recovery
- Soft delete → `DELETED` state (recoverable within grace period)
- Hard delete → GC removes bytes (irrecoverable)

---

## Caching Strategy

| Cache Layer | What | TTL | Tool |
|-------------|------|-----|------|
| Client / browser | GET response + ETag | Via `Cache-Control` header | Browser |
| CDN | Popular public objects | Minutes to days | CloudFront, Fastly |
| Front-end server | Partition map (account+bucket → chunk servers) | 60s | In-process |
| Metadata cache | Object metadata (location, size, ACL) | 30s | Redis |
| Chunk server | Hot chunk data | LRU, fit in RAM | In-process |

**Cache invalidation**: on write/delete, invalidate metadata cache key `(account, bucket, object_key)`.
ETag + `If-None-Match` header for conditional GETs — return 304 if object unchanged.

---

## S3 vs Azure Blob vs GCS — Design Differences

| Aspect | Amazon S3 | Azure Blob Storage | Google Cloud Storage |
|--------|-----------|-------------------|---------------------|
| **Namespace** | account → bucket (globally unique) → object key | account → container → blob | project → bucket → object |
| **Subfolder** | Key prefix + delimiter (virtual) | Virtual directory (prefix convention) | Key prefix + delimiter (virtual) |
| **Consistency** | Strong (single-region) | Strong | Strong |
| **Storage tiers** | Standard, IA, Glacier Instant, Glacier, Deep Archive | Hot, Cool, Cold, Archive | Standard, Nearline, Coldline, Archive |
| **Max object size** | 5 TB (multipart) | 4.75 TB (block blob) | 5 TB |
| **Multipart** | 5 MB min part, 10k max parts | Block blobs: 50k blocks × 4000 MB | Compose (32 parts max) |
| **Object types** | Single type (object) | Block blob / Append blob / Page blob | Single type (object) |
| **Append object** | No native append | Append blob — atomic append, no overwrite | No native append |
| **Versioning** | Optional, per-bucket | Optional, per-container | Optional, per-bucket |
| **Cross-region replication** | CRR (Cross-Region Replication) | GRS / GZRS | Multi-region buckets |
| **Event notifications** | S3 Events → SNS / SQS / Lambda / EventBridge | Event Grid → Azure Functions | Pub/Sub |
| **Direct upload** | Pre-signed URL | SAS Token | Signed URL |

---

## Lifecycle Management (Tiering)

```
New object    → Standard tier          (low latency, high cost)
After 30 days → Infrequent Access      (slightly higher retrieval latency)
After 90 days → Glacier Instant        (ms retrieval, cheaper)
After 1 year  → Glacier / Archive      (minutes–hours retrieval, very cheap)
After 7 years → Expire / Delete
```

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Namespace | account → bucket → object key with prefix convention | Flat is simpler; virtual subfolders via prefix give UX without FS complexity |
| Chunk size | 64 MB | Balance between metadata overhead and I/O efficiency |
| Replication | 3× sync in-region, async cross-region | Durability without blocking on geo latency |
| Cold data | Erasure coding (6,3) | 1.5× overhead vs 3×; same 11-nine durability |
| Metadata store | Cassandra / DynamoDB (partitioned by account+bucket) | Scale to billions of objects |
| Delete | Soft delete + GC | Accidental delete recovery + atomic reference cleanup |
| Large upload | Multipart | Parallelism + resumability |
| Client upload | Pre-signed URL (direct to storage) | Server never handles bytes → massive throughput |
| Consistency | Strong single-region, eventual cross-region | Practical balance of latency vs correctness |
