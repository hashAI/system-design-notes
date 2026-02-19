# Design a Video Streaming System (basic scaling ideas) — full system design

Video streaming systems are dominated by two realities:

- **Bandwidth is huge** (playback is expensive)
- **Processing is heavy** (transcoding is CPU/GPU intensive)

This design covers a practical “YouTube/Netflix-lite” architecture:

- upload
- transcode
- store segments
- deliver via CDN
- support adaptive bitrate playback (ABR)

---

## 1) Requirements

### Functional requirements

- Upload videos
- Transcode into multiple qualities (e.g., 240p/480p/720p/1080p)
- Playback on web/mobile
- Adaptive bitrate streaming (ABR) using HLS/DASH
- Basic metadata (title, description)

### Nice-to-have

- Live streaming
- DRM / protected content
- Subtitles
- Recommendations/search
- Analytics (watch time)

### Non-functional requirements

- Global low-latency playback
- High availability
- Cost efficiency (storage + CDN egress)
- Durable storage (don’t lose uploads)
- Secure access controls (private videos)

---

## 2) Back-of-the-envelope scale (example)

Assume:

- 1M DAU
- 10 minutes/day watched per user
- average bitrate 3 Mbps

Daily egress:

- 1M × 10 min × 3 Mbps
- = 1M × 600s × 3 Mbps = 1.8e9 Mb/day
- = 225,000 GB/day ≈ 225 TB/day

Key takeaway:

- playback bandwidth is the main cost
- CDN is mandatory

Uploads:

- much smaller than playback, but require reliable ingestion and heavy processing.

---

## 3) High-level architecture

### Components

- **Upload service**:
  - auth, metadata, presigned upload URLs
- **Object storage**:
  - raw uploads
  - transcoded segments
  - manifests
- **Transcoding pipeline**:
  - queue (Kafka/SQS/RabbitMQ)
  - worker fleet (CPU/GPU)
  - produces HLS/DASH outputs
- **CDN**:
  - serves manifests and segments
- **Playback API**:
  - returns playback URLs, checks permissions
- **Metadata DB**:
  - video records, processing status, variants
- **Analytics pipeline** (optional):
  - client events → stream → aggregation

---

## 4) Upload flow (control plane + data plane)

### Step 1: Initiate upload

Client → Upload API:

- create a video record with status `uploading`
- return presigned URL(s)

### Step 2: Upload bytes

Client uploads directly to object storage (or upload gateway) using presigned URLs.

Why presigned:

- avoids proxying huge bandwidth through your API servers

### Step 3: Finalize upload

Client calls “complete upload.”

System:

- marks status `uploaded`
- enqueues transcode job

---

## 5) Transcoding pipeline

Transcoding is async and can take time.

### Worker responsibilities

- download raw video
- transcode into multiple renditions
- generate:
  - HLS `.m3u8` manifests and `.ts` segments, or
  - DASH `.mpd` manifests and `.m4s` segments
- upload segments + manifests to object storage
- update metadata DB with variant locations

### Scaling

- autoscale workers based on queue depth
- use GPU workers for high throughput if needed

### Failure handling

- retry with backoff
- DLQ for bad/corrupted inputs

---

## 6) Playback flow (adaptive bitrate streaming)

### How ABR works (simple)

1. Player downloads a **manifest** (playlist) that lists available qualities.
2. Player downloads small **segments** (e.g., 2–6 seconds each).
3. Player picks quality based on current bandwidth and buffer health.

Why segments:

- easier to cache
- enables seeking
- enables quality switching smoothly

---

## 7) CDN strategy

CDN is essential:

- segments are static and cacheable
- traffic is global

Best practices:

- long TTL for segments (content-addressed paths)
- versioned URLs so you don’t need invalidations
- range requests supported

Cache key correctness:

- if content is public, caching is easy
- if private, use signed URLs/cookies and avoid leaking private assets

---

## 8) Storage model

Store:

- raw upload (for re-transcoding later)
- transcoded variants
- manifests

Key cost control:

- lifecycle policies:
  - keep raw uploads for X days unless needed
  - keep popular variants longer

---

## 9) Metadata model

Example tables:

- `videos(video_id, owner_id, title, status, created_at, visibility, ...)`
- `video_variants(video_id, variant_id, resolution, bitrate, codec, manifest_url, ...)`
- `video_segments(video_id, variant_id, segment_prefix_url, segment_count, ...)` (often implicit)

Status progression:

- `uploading` → `uploaded` → `processing` → `ready` (or `failed`)

---

## 10) Protected content (signed URLs and DRM)

Simple protection:

- Playback API checks permissions and returns **signed URLs** (short expiry).

DRM (advanced):

- key management, license server, encrypted segments

Interview-friendly:

- mention DRM as optional and complex; start with signed URLs.

---

## 11) Live streaming (optional extension)

Differences:

- segments are produced continuously
- latency constraints are tighter
- ingest uses RTMP/SRT/WebRTC depending on use case

Architecture:

- ingest servers → transcode → segmenter → CDN

---

## 12) Analytics (optional)

Client emits events:

- play, pause, seek
- buffer events
- bitrate switches
- watch time

Pipeline:

- events → stream → aggregation → dashboards

This helps:

- quality monitoring
- recommendations
- CDN optimization

---

## 13) Failure modes

- Transcoding backlog:
  - videos take longer to become ready
  - show “processing” state
- CDN outage in one region:
  - fallback to another CDN/region (multi-CDN if needed)
- Storage outage:
  - playback fails; mitigate with multi-region replication for popular content

---

## 14) Interview-ready summary (30 seconds)

“We ingest uploads via an API that returns presigned URLs so clients upload directly to object storage. An async transcode pipeline (queue + worker fleet) produces HLS/DASH manifests and segmented renditions at multiple bitrates, stores them in object storage, and serves them globally through a CDN. Playback fetches a manifest then segments, using adaptive bitrate selection. We protect private content with signed URLs, monitor transcode lag and playback QoE metrics, and control cost with lifecycle policies and caching.”

