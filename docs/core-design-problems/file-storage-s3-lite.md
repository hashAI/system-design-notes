# Design a File Storage System (S3-lite) — full system design

Object storage (like Amazon S3) is the backbone of modern systems:

- photos and videos
- backups
- documents
- logs and data lakes

This design builds an “S3-lite” with the core ideas:

- upload/download objects
- metadata vs data plane separation
- durability via replication / erasure coding
- presigned URLs for scalable uploads

---

## 1) Requirements

### Functional requirements

- Buckets and objects:
  - create bucket
  - put object (upload)
  - get object (download)
  - delete object
  - list objects by prefix (optional)
- Object metadata:
  - size, content-type, checksum, created_at
  - ACL (private/public) (simple)
- Support large objects (multipart upload)

### Non-functional requirements

- Very high durability (e.g., “11 9s” is S3; we can target “very high”)
- High availability
- High throughput for uploads and downloads
- Cost efficiency (storage is huge)
- Security: auth, encryption, access control, audit

### Out of scope (unless asked)

- Complex lifecycle rules (tiering)
- Cross-region replication policies
- Full IAM-like permission system

---

## 2) Scale estimation (example)

Assume:

- 10M users
- 2 uploads/day/user average
- average object size 2 MB

Uploads/day:

- 20M objects/day
- data/day = 40 TB/day

Within 30 days:

- 1.2 PB raw (before replication)

Key implication:

- data storage dominates cost
- metadata is relatively small but needs strong consistency

---

## 3) High-level architecture: control plane vs data plane

This separation is the most important object storage concept.

### Control plane (metadata operations)

- handles:
  - auth
  - bucket/object metadata CRUD
  - generating presigned URLs
  - ACL checks
- uses a strongly consistent metadata store (SQL)

### Data plane (bytes transfer)

- handles:
  - upload/download of object bytes
  - chunk storage and replication/erasure coding
- optimized for throughput

Why split?

- metadata ops are small and transactional
- data ops are huge and bandwidth-heavy

---

## 4) APIs (simple version)

### Create bucket

`POST /v1/buckets { name }`

### Start upload (presigned)

`POST /v1/buckets/{bucket}/objects/{key}:initiate`

Response:

- presigned URL(s) for upload
- upload_id (for multipart)

### Complete upload

`POST /v1/buckets/{bucket}/objects/{key}:complete { upload_id, parts[] }`

### Download (presigned)

`POST /v1/buckets/{bucket}/objects/{key}:download-url`

Response:

- presigned GET URL

### Get metadata

`GET /v1/buckets/{bucket}/objects/{key}:meta`

---

## 5) Metadata model

Tables (simplified):

- `buckets(bucket_id, name, owner_id, created_at)`
- `objects(object_id, bucket_id, key, version, size, checksum, content_type, created_at, deleted_at?)`
- `object_parts(object_id, part_no, chunk_id, size, checksum)`
- `acls(object_id, principal, permission)` (optional)

Important:

- metadata must be consistent (you don’t want “dangling” objects).
- listing by prefix needs careful indexing (or separate index structure).

---

## 6) Data storage: chunks, replication, erasure coding

Objects are stored as **chunks**:

- split object into fixed-size chunks (e.g., 8MB)
- store chunks on storage nodes

### Option A: Replication (simplest)

- store each chunk on 3 nodes (3× replication)

Pros:

- simple reads and repairs
- fast recovery

Cons:

- high storage overhead (3×)

### Option B: Erasure coding (more efficient)

- split chunk into data blocks + parity blocks (e.g., 10+4)
- can lose some blocks and still recover

Pros:

- lower overhead than replication

Cons:

- more CPU
- more complex repair
- slower small reads/writes

Interview-friendly approach:

- start with replication, mention erasure coding for cost optimization later.

---

## 7) Consistent placement (where chunks go)

You need to distribute chunks across nodes evenly:

- consistent hashing ring
- or a placement service that chooses nodes based on:
  - capacity
  - rack/AZ diversity
  - current load

Durability requires **failure domain awareness**:

- replicas should be in different racks / AZs

---

## 8) Upload flow (presigned URLs)

Why presigned?

- If your API servers proxy all uploads, they become bandwidth bottlenecks.

Flow:

1. Client calls control plane to initiate upload.
2. Control plane authenticates and returns presigned URLs to storage nodes (or an upload gateway).
3. Client uploads directly to storage nodes.
4. Client completes upload; control plane writes final metadata and makes object visible.

Important correctness detail:

- object should become visible only after all parts are uploaded and checksummed.

---

## 9) Download flow (CDN + range requests)

Downloads are bandwidth heavy.

Approach:

- serve via CDN where possible
- support range requests:
  - video players and resumable downloads rely on this

Flow:

1. Client requests download URL.
2. Client downloads from CDN/storage.

---

## 10) Garbage collection (deletes and abandoned uploads)

Deletes are often “logical” first:

- mark object deleted in metadata (tombstone)
- background GC deletes chunks later

Why:

- deleting chunks immediately can be expensive
- it’s safer under retries and partial failures

Abandoned multipart uploads:

- have TTLs; GC cleans them up.

---

## 11) Consistency model

Metadata should be strongly consistent:

- if `PUT` returns success, `GET meta` should reflect it

Data plane can be eventually consistent internally during replication/repair, as long as you maintain durability guarantees.

---

## 12) Failure handling

### Storage node failure

- replication ensures copies exist
- background repair re-replicates missing chunks

### Network partitions

- uploads may partially succeed; multipart + checksums help safely resume

### Corruption

- store checksums per chunk/part
- verify on read; if corrupted, fetch another replica and repair

---

## 13) Security

- Authn/authz at control plane
- Presigned URLs expire quickly
- Encrypt in transit (TLS)
- Encrypt at rest (server-side encryption) (optional for S3-lite but mention)
- Audit logs for object access (important in enterprise)

---

## 14) Observability

Metrics:

- upload/download throughput
- error rates by operation
- storage capacity utilization
- repair queue length and time to heal
- hot objects and cache hit ratio (if CDN)

---

## 15) Interview-ready summary

“We split the system into a control plane for metadata/auth and a data plane for bytes transfer. Objects are stored as chunks on storage nodes with replication across failure domains for durability, and checksums for corruption detection. Clients upload/download directly via presigned URLs to avoid proxy bottlenecks, with multipart upload for large files. Deletes are tombstoned in metadata and garbage-collected asynchronously. We monitor throughput, error rates, capacity, and repair health.”

