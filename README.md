## System Design Study Guide (Senior SWE – Product Companies)

This repo is a single-source, interview-oriented **system design handbook**. It’s written for senior software engineers preparing for product-company interviews: practical tradeoffs, scaling paths, reliability thinking, and the “how to talk about it” structure.

If you deeply understand the fundamentals and can walk through the 15 core designs below, you’re ready.

### How to use this guide

- **First pass (breadth)**: read the interview framework + fundamentals end-to-end.
- **Second pass (depth)**: pick 1 design problem/day; write your own design doc using the provided template.
- **Third pass (fluency)**: do timed mocks: 5 min requirements + 10 min high-level + 15 min deep dives.

### Table of contents

- **Interview framework**
  - [What interviewers evaluate](docs/interview-framework/what-interviewers-evaluate.md)
  - [Design template (copy/paste)](docs/interview-framework/design-template.md)
  - [Back-of-the-envelope math (capacity estimation)](docs/interview-framework/capacity-estimation.md)
- **Fundamentals (non‑negotiable)**
  - [Load balancing](docs/fundamentals/load-balancing.md)
  - [Horizontal vs vertical scaling](docs/fundamentals/horizontal-vs-vertical-scaling.md)
  - [Caching (Redis concepts, eviction, invalidation)](docs/fundamentals/caching.md)
  - [Database indexing](docs/fundamentals/database-indexing.md)
  - [SQL vs NoSQL (when and why)](docs/fundamentals/sql-vs-nosql.md)
  - [Replication & sharding](docs/fundamentals/replication-and-sharding.md)
  - [CAP theorem (practical)](docs/fundamentals/cap-theorem.md)
  - [Consistency models](docs/fundamentals/consistency-models.md)
  - [Rate limiting](docs/fundamentals/rate-limiting.md)
  - [Message queues (Kafka/RabbitMQ concepts)](docs/fundamentals/message-queues.md)
  - [CDN basics](docs/fundamentals/cdn-basics.md)
- **Core design problems (practice these 15)**
  - [URL shortener](docs/core-design-problems/url-shortener.md)
  - [Rate limiter](docs/core-design-problems/rate-limiter.md)
  - [Chat system](docs/core-design-problems/chat-system.md)
  - [Notification system](docs/core-design-problems/notification-system.md)
  - [News feed / timeline](docs/core-design-problems/news-feed.md)
  - [File storage (S3-lite)](docs/core-design-problems/file-storage-s3-lite.md)
  - [API gateway](docs/core-design-problems/api-gateway.md)
  - [Payment system (high-level flow)](docs/core-design-problems/payment-system.md)
  - [Logging & monitoring system](docs/core-design-problems/logging-and-monitoring-system.md)
  - [Search autocomplete](docs/core-design-problems/search-autocomplete.md)
  - Video streaming (basic scaling ideas)
  - Distributed cache
  - Job scheduler
  - E-commerce system
  - Ride-sharing matching (high level)

---

## Interview framework

### What interviewers evaluate

- **Requirements clarity**: ask the right questions; define scope.
- **Correctness**: does the system do what it claims under failures and concurrency?
- **Tradeoffs**: latency vs consistency, cost vs reliability, simplicity vs flexibility.
- **Scalability**: can it grow \(10×–100×\) without a rewrite?
- **Reliability**: timeouts, retries, backpressure, overload behavior, disaster recovery.
- **Data thinking**: data model, queries, indexes, partitions, retention, privacy.
- **Operability**: metrics/logs/traces, alerting, debugging, deployment safety.

### The system design template (copy/paste)

Use this structure in every interview. It keeps you coherent and makes tradeoffs explicit.

- **1) Requirements**
  - **Functional**: core flows + edge cases (create/read/update/delete; realtime vs batch).
  - **Non-functional**: latency (p50/p95/p99), availability, durability, consistency, compliance, cost.
  - **Out of scope**: explicitly list what you’re not building.
- **2) Scale estimate**
  - DAU/MAU, QPS, peak factor, read/write ratio
  - bandwidth, storage growth, retention
- **3) APIs**
  - request/response shape, pagination, idempotency keys, error codes
- **4) Data model**
  - key entities, relationships, access patterns, indexes
- **5) High-level architecture**
  - main components and how they connect
  - synchronous vs async boundaries
- **6) Deep dives**
  - caching strategy, partitioning strategy, replication
  - consistency choices and their UX impact
  - failure modes + mitigations
- **7) Observability & operations**
  - metrics, logs, traces, dashboards, alerts
  - deploy strategy, rollbacks, migrations
- **8) Security & abuse**
  - authn/authz, PII, encryption, rate limits, fraud/spam controls

### Back-of-the-envelope math (capacity estimation)

You don’t need perfect numbers; you need reasonable orders of magnitude and bottleneck intuition.

- **QPS**: \( \text{QPS} \approx \frac{\text{requests/day}}{86{,}400} \). Multiply by a **peak factor** (often 5–20×).
- **Bandwidth**: \( \text{bytes/request} \times \text{QPS} \).
- **Storage**:
  - events/day × bytes/event × retention days
  - add replication factor and indexes overhead
- **Latency budgets**: for p99 targets, budget per hop (LB, app, cache, DB) and budget for retries.

Rules of thumb:

- Aim for **stateless app tiers** so you can scale horizontally.
- Prefer **idempotent operations** so retries are safe.
- Use **timeouts + retries + circuit breakers** (and cap retries to avoid retry storms).

---

## Fundamentals (non‑negotiable)

### Load balancing

**Goal**: distribute traffic across instances to improve throughput, reduce latency, and survive failures.

- **Where LBs live**
  - **L4** (TCP/UDP): fast, opaque; good for raw throughput and generic services.
  - **L7** (HTTP/gRPC): route by path/host/header; supports TLS termination, auth, retries, canary.
- **Common algorithms**
  - **Round robin / weighted RR**
  - **Least connections / least outstanding requests**
  - **Consistent hashing** (important for caches and sticky data)
- **Health checks**
  - **Liveness**: “process is up”
  - **Readiness**: “can serve traffic” (e.g., warmed caches, dependencies reachable)
- **Sticky sessions**
  - Avoid when possible (prefer stateless + shared session store), but sometimes used for legacy.
- **Global traffic**
  - DNS-based, Anycast, geo routing, active-active vs active-passive.
- **Failure/overload behavior**
  - Connection draining, load shedding (return 429/503), priority queues, brownouts.

### Horizontal vs vertical scaling

- **Vertical**: bigger machine (more CPU/RAM/IOPS). Easy but has a ceiling; may increase blast radius.
- **Horizontal**: more machines. Requires stateless services, partitioning, coordination strategies.
- **A key senior skill**: recognize when you must change the data model/partitioning to scale, not just add servers.

### Caching (Redis concepts, eviction, invalidation)

**Why cache**: reduce latency and backend load. **Risk**: stale data, thundering herds, correctness bugs.

- **Cache layers**
  - **Client** (browser/app), **CDN/edge**, **service cache**, **distributed cache**, **DB buffer cache**.
- **Common patterns**
  - **Cache-aside**: app reads cache; on miss read DB then fill cache.
    - Pros: simple; Cons: cold misses; must handle stampede.
  - **Write-through**: write cache + DB synchronously.
    - Pros: consistent cache; Cons: higher write latency.
  - **Write-behind**: write cache then async flush to DB.
    - Pros: fast writes; Cons: risk of data loss/complexity.
- **Invalidation strategies**
  - **TTL-based** (eventual correctness)
  - **Explicit invalidation** (publish invalidation events; delete keys after DB write)
  - **Versioned keys** (include version in key to avoid deleting)
- **Eviction**
  - Typical policies: **LRU**, **LFU**, random; and Redis variants like allkeys/volatile LRU/LFU.
  - Understand hot keys and memory fragmentation; keep value sizes bounded.
- **Stampede / dogpile protection**
  - Request coalescing (“single flight”), soft TTL + background refresh, jittered TTLs.
  - Use locks carefully; avoid global locks.
- **Redis fundamentals (what to know)**
  - Data types: strings, hashes, sets, sorted sets, streams.
  - Persistence options (RDB/AOF), replication, Sentinel, Cluster sharding.
  - Atomic operations (INCR, Lua scripts) for counters/rate limiting.

### Database indexing

- **What an index is**: a secondary structure to speed up reads at the cost of write overhead and storage.
- **B-Tree indexes**
  - Great for range queries, ordering, prefix matches on composite keys.
- **Composite indexes**
  - Order matters: index on (a, b) supports queries by a and (a,b), not just b.
- **Covering indexes**
  - If the index contains all needed columns, DB can avoid table lookup.
- **Tradeoffs**
  - More indexes: faster reads, slower writes, more storage, more maintenance during compaction/vacuum.
- **How seniors debug**
  - Use `EXPLAIN` / query plans, understand selectivity/cardinality, fix “full table scans”.

### SQL vs NoSQL (when and why)

Think in terms of **data model + access patterns + consistency needs**.

- **SQL / relational**
  - Best for: strong consistency, transactions, complex joins, ad-hoc queries, invariants.
  - Scaling: vertical + read replicas; sharding is possible but adds complexity (cross-shard joins/transactions).
- **Key-value (Redis, DynamoDB)**
  - Best for: simple access patterns (get/put by key), massive scale, low latency.
- **Document (MongoDB)**
  - Best for: flexible schema, nested objects, moderate joins (or embedding).
  - Watch out for: unbounded document growth; multi-document transactions cost/complexity.
- **Wide-column (Cassandra)**
  - Best for: write-heavy, time-series-like access patterns; design by query.
  - Tradeoff: weaker consistency and limited query flexibility.
- **Search index (Elasticsearch/OpenSearch)**
  - Best for: full-text search, aggregations; not a source of truth for transactions.

Rule: **Choose the simplest store that supports your invariants**. Add specialized stores (cache/search) as derived views.

### Replication & sharding

- **Replication** (copies)
  - **Leader-follower**: strong-ish writes on leader; reads can scale on followers (but face replication lag).
  - **Multi-leader**: better multi-region writes; conflict resolution complexity.
  - **Quorum**: tune \(R/W/N\) to trade consistency vs availability (common in Dynamo-style systems).
- **Sharding** (partitioning data)
  - Strategies: **hash**, **range**, **directory-based**.
  - Problems: hot shards, resharding, cross-shard queries, distributed transactions.
  - Mitigations: consistent hashing, virtual nodes, adaptive rebalancing, “scatter-gather” only when necessary.

### CAP theorem (practical understanding)

During a network partition you must choose between:

- **Consistency (C)**: every read sees the latest write (or errors).
- **Availability (A)**: every request gets a non-error response (may be stale).

In real systems:

- You often build **CP for critical invariants** (payments, inventory) and allow partial unavailability.
- You build **AP for feed/analytics/telemetry** where stale/approximate is acceptable.
- You can degrade gracefully: serve from cache, serve stale reads, or reduce features (“brownout”).

### Consistency models (strong vs eventual)

- **Strong / linearizable**: behaves like a single up-to-date copy (harder, more latency).
- **Eventual**: replicas converge; reads may be stale.
- **Useful session guarantees**
  - **Read-your-writes**: after you write, you see your own write.
  - **Monotonic reads**: you don’t go backwards in time.
  - **Causal consistency**: preserves cause-effect order for related operations.

Tie to product UX: chat “sent” indicators, feed freshness, inventory accuracy, etc.

### Rate limiting

**Goal**: protect systems from abuse and overload; enforce fairness and quotas.

- **Where to enforce**
  - Edge/CDN, API gateway, service-level, per-function or per-resource.
- **Algorithms**
  - **Token bucket**: allows bursts; common for APIs.
  - **Leaky bucket**: smooths traffic; good for steady throughput.
  - **Fixed window**: simple but bursty at window edges.
  - **Sliding window**: more accurate; more state.
- **Distributed rate limiting**
  - Central store (Redis) with atomic ops (INCR + TTL; Lua for token bucket).
  - Local + periodic reconciliation for lower latency (accept small inaccuracies).
- **Good API behavior**
  - Return 429, include `Retry-After`, rate limit headers, and clear error messages.

### Message queues (Kafka/RabbitMQ concepts)

**Why**: decouple producers/consumers, smooth spikes, enable async work and event-driven architecture.

- **RabbitMQ (classic broker)**
  - Great for: task queues, per-message routing patterns, low-latency work distribution.
  - Concepts: exchanges, queues, routing keys, acknowledgements, prefetch.
- **Kafka (distributed log)**
  - Great for: event streams, high throughput, replay, multiple independent consumers.
  - Concepts: topics, partitions, offsets, consumer groups, retention, compaction.
- **Delivery semantics**
  - **At-most-once**: may drop.
  - **At-least-once**: may duplicate (most common).
  - **Exactly-once**: possible but complex; usually achieved with idempotency + transactional writes.
- **Senior-level must-haves**
  - Idempotent consumers, retry policies, dead-letter queues, backpressure, schema evolution.

### CDN basics

**Why**: reduce latency and origin load for static and cacheable content.

- **Key concepts**
  - Edge POPs, origin, cache keys, TTLs, cache-control, invalidation/purge.
  - **Signed URLs/cookies** for private content.
  - **Range requests** for large files/video.
- **Dynamic acceleration**
  - TLS termination, connection reuse, edge compute, request collapsing.
- **Pitfalls**
  - Cache poisoning, incorrect cache keys (e.g., missing auth/user dimension), stale content.

---

## Core design problems (practice these 15)

For each problem, aim to articulate: **requirements → APIs → data model → architecture → scaling → failure modes → tradeoffs**.

### 1) URL shortener

#### Requirements

- **Functional**: create short URL, redirect, custom alias, expiration, basic analytics (click count).
- **Non-functional**: very low redirect latency (p99), high read QPS, high availability.

#### APIs

- `POST /v1/shorten { long_url, custom_alias?, expire_at? } -> { short_url }`
- `GET /{code} -> 301/302 redirect`
- `GET /v1/analytics/{code} -> { clicks, last_clicked_at, ... }`

#### Data model

- `url_mapping(code PK, long_url, created_at, expire_at, owner_id?, checksum?)`
- Optional: `click_events(code, ts, ip_hash, user_agent_hash)` (or aggregated counters).

#### Architecture

- **Redirect path**: CDN/edge cache → LB → stateless redirect service → cache (Redis) → DB.
- **Write path**: shorten service → code generation → DB write → cache fill.

#### Key decisions / tradeoffs

- **Code generation**
  - Base62 of an auto-increment ID (needs a reliable ID generator; predictable codes)
  - Random codes (needs collision handling; not guessable)
  - Snowflake-like IDs + encoding (good distribution)
- **Redirect 301 vs 302**
  - 301 caches aggressively (good for stable links), 302 for mutable targets.
- **Hot keys**
  - Very popular short URLs: cache heavily; consider edge caching with correct cache-control.
- **Analytics**
  - Don’t write synchronously on redirect. Emit async events to Kafka; aggregate in background.

#### Failure modes

- Cache down: fall back to DB (and shed load if DB is at risk).
- DB down: redirects can still work for cached popular links; writes fail.

---

### 2) Rate limiter (as a service / gateway)

#### Requirements

- **Functional**: per-user/per-IP/per-route limits, burst handling, distributed enforcement.
- **Non-functional**: low latency (added overhead must be small), high availability, correctness “good enough”.

#### Architecture

- **At edge/API gateway**: check limit before forwarding.
- **Store**: Redis (cluster) for counters/tokens; Lua script for atomicity.
- **Fallback**: local in-memory limiter if Redis is unavailable (accept approximate behavior).

#### Algorithms (when to choose)

- Token bucket for APIs (bursty but bounded).
- Leaky bucket to smooth processing to downstream systems.
- Concurrency limits for expensive endpoints (protect DB).

#### Pitfalls

- Hot users/keys causing Redis hotspots → shard by `(user_id, route)` and use Redis Cluster.
- Clock skew when using window-based methods → prefer token bucket state.

---

### 3) Chat system (1:1 + group)

#### Requirements

- **Functional**: send/receive messages, online presence, typing indicators (optional), offline delivery, read receipts (optional).
- **Non-functional**: low latency for online delivery, durability for messages, scalable fanout.

#### Architecture (high-level)

- **Realtime connections**: WebSocket gateways (stateless, scaled horizontally).
- **Message service**: validates, assigns IDs, persists, publishes events.
- **Storage**
  - Messages: wide-column store (e.g., Cassandra) or sharded SQL for message history.
  - Metadata: users, conversations, memberships in SQL.
- **Event streaming**: Kafka for message events; consumers push to online users and enqueue push notifications.

#### Ordering & delivery

- Use **per-conversation sequencing** (monotonic sequence number) to reason about ordering.
- Delivery semantics: typically **at-least-once**; clients must dedupe by message ID.
- Read receipts: stateful and privacy-sensitive; treat as metadata updates.

#### Presence

- Presence service with heartbeat; store ephemeral state in Redis with TTL.
- Avoid global fanout storms; batch presence updates.

#### Failure modes

- WebSocket node dies: clients reconnect; resume from last seen sequence.
- Kafka lag: online delivery may delay; ensure persistence is the source of truth.

---

### 4) Notification system (email/SMS/push/in-app)

#### Requirements

- **Functional**: send notifications, templates, user preferences, scheduling, retries, dedupe.
- **Non-functional**: high reliability, provider failover, idempotency.

#### Architecture

- API → validate & persist notification request → enqueue event (Kafka/RabbitMQ) → worker pool.
- Workers:
  - render templates
  - apply preferences/quiet hours
  - send via providers (email/SMS/push)
  - record delivery outcomes
- Use **DLQ** for poisoned messages and **retry with backoff** for transient provider failures.

#### Idempotency & dedupe

- Require an idempotency key per request (e.g., `notification_id`).
- Deduplicate by `(user_id, template_id, event_id)` if needed.

#### Observability

- Success rate per provider, latency, bounce/complaint rates, queue lag, retry counts.

---

### 5) News feed / timeline

#### Requirements

- **Functional**: post content, follow users, fetch ranked feed, pagination, hide/report (optional).
- **Non-functional**: low-latency feed reads, scalable fanout, freshness tradeoffs.

#### Two canonical approaches

- **Fanout-on-write (push model)**
  - On post: push post ID into each follower’s inbox.
  - Pros: fast reads; Cons: expensive writes for high-fanout users.
- **Fanout-on-read (pull model)**
  - On read: fetch recent posts from followed users and merge/rank.
  - Pros: cheap writes; Cons: slow reads, heavy compute.

Most real systems use a hybrid:

- Push for most users, pull for celebrity/high-fanout accounts.
- Precompute candidate sets; rank with ML separately.

#### Storage & caching

- Store posts in primary DB; store per-user feed inbox as list of post IDs (KV or wide-column).
- Cache first page aggressively; handle pagination with cursors (timestamp + ID).

#### Consistency

- Eventual consistency is acceptable; define UX guarantees (e.g., “your own posts appear immediately”).

---

### 6) File storage (S3-lite)

#### Requirements

- **Functional**: upload/download objects, metadata, multipart uploads, access control, delete, versioning (optional).
- **Non-functional**: very high durability, high throughput, cost efficiency.

#### Architecture

- **API service**: auth, metadata ops, presigned URLs.
- **Data path**: clients upload directly to storage nodes via presigned URLs (avoid proxying through app).
- **Storage nodes**: store chunks; replicate or erasure-code; background repair.
- **Metadata store**: strongly consistent DB for object metadata (bucket, key, version, ACL, chunk pointers).

#### Durability strategies

- Replication (3×) is simple but costly.
- Erasure coding reduces storage overhead but increases CPU/latency and repair complexity.

#### Hot objects & CDN

- Use CDN for downloads; use range requests for large objects.
- Separate metadata from data so listing doesn’t touch large blobs.

---

### 7) API gateway

#### Responsibilities

- Routing, authn/authz, TLS termination
- Rate limiting, quotas
- Request/response transformation
- Retries/timeouts/circuit breakers
- Observability (metrics/logs/traces)

#### Design points

- Keep it stateless; config via control plane (push) or service discovery.
- Support canary releases and traffic shadowing for safe rollout.
- Do not hide all failures with retries; ensure idempotency and avoid retry storms.

---

### 8) Payment system (basic high-level flow)

#### Requirements

- **Functional**: authorize, capture, refund, webhook handling, reconciliation.
- **Non-functional**: correctness, auditability, idempotency, fraud controls, compliance (PCI).

#### Core concepts

- **Idempotency keys** on every client-initiated payment mutation.
- **Ledger**: append-only, double-entry bookkeeping model for money movement.
- **State machine**: `created -> authorized -> captured -> settled` (+ failure states).

#### Architecture

- Payment API → validate → write payment intent + ledger entries → call PSP (Stripe/Adyen/etc.) → record outcome → emit events.
- Webhooks are **untrusted**: verify signatures, dedupe, and treat as eventual confirmation.

#### Failure modes

- Network timeout after charging: must reconcile via PSP APIs and idempotency to avoid double charge.
- Partial failure: ensure the ledger is the source of truth; all derived views are rebuildable.

---

### 9) Logging & monitoring system

#### Requirements

- **Functional**: ingest logs/metrics/traces, search and dashboards, alerts, retention policies.
- **Non-functional**: high ingestion throughput, cost-aware storage, secure multi-tenant access.

#### Architecture (typical)

- **Agents** (on hosts) → ingestion endpoints → queue (Kafka) → processing (parsing/enrichment) → storage.
- Storage split:
  - **Logs**: object storage (cheap) + index (Elasticsearch/OpenSearch) for search
  - **Metrics**: TSDB (Prometheus remote write, Cortex/Mimir, or similar)
  - **Traces**: sampled storage + index; correlate via trace/span IDs

#### Key tradeoffs

- Full indexing is expensive. Use tiering: hot indexed window + cold archived logs.
- Sampling for traces; dynamic sampling for high-volume services.

---

### 10) Search autocomplete

#### Requirements

- **Functional**: prefix suggestions, typo tolerance (optional), personalization (optional), very low latency.
- **Non-functional**: p95 in tens of ms, high QPS.

#### Approaches

- **Trie / prefix tree** (in-memory) for top queries; update periodically.
- **Search engine** (Elasticsearch) with edge n-grams and popularity scoring.
- **Hybrid**
  - serve cached/trie results for top prefixes
  - fallback to search engine for long-tail

#### Data pipeline

- Collect queries + clicks → aggregate popularity by locale/device → publish model → rebuild trie.

---

### 11) Video streaming (basic scaling ideas)

#### Requirements

- **Functional**: upload, transcode, playback, adaptive bitrate, DRM (optional), live (optional).
- **Non-functional**: huge bandwidth, global low latency, cost constraints.

#### Architecture

- Upload to object storage → async transcode pipeline (queue + workers) → produce HLS/DASH segments → store segments → CDN distribution.
- Player fetches a manifest and then segments; adaptive bitrate selects quality.

#### Key scaling points

- Bandwidth is dominated by playback; CDN is mandatory.
- Transcoding is CPU/GPU heavy; autoscale workers; prioritize popular content.
- Use signed URLs/token auth for protected content; enforce geo/licensing at edge when possible.

---

### 12) Distributed cache

#### Requirements

- **Functional**: shared cache for many services, predictable latency, high availability.
- **Non-functional**: avoid hotspots, keep tail latency low, simple operations.

#### Architecture choices

- **Client-side consistent hashing**
  - Pros: no extra hop; Cons: client complexity; membership changes require coordination.
- **Proxy-based**
  - Pros: simpler clients; Cons: proxy adds hop and can be a bottleneck.

#### Key concerns

- Hot keys: replicate hot keys or split value.
- Consistency: caches are typically best-effort; design for stale reads.
- Cache coherence: avoid trying to make cache “strongly consistent”; use explicit invalidations where needed.

---

### 13) Job scheduler

#### Requirements

- **Functional**: run jobs now/later (delayed), cron-like schedules, retries, concurrency control, visibility.
- **Non-functional**: at-least-once execution, high reliability, operational tooling.

#### Architecture

- API → persist job definition → scheduler loop picks due jobs → enqueue to worker queue → workers execute.
- Use leases/heartbeats to detect stuck workers and requeue.

#### Execution semantics

- Prefer **at-least-once** plus **idempotent job handlers**.
- DLQ for repeated failures; exponential backoff; poison-pill detection.

---

### 14) E-commerce system

#### Requirements

- **Functional**: catalog, search, cart, checkout, orders, payments, inventory, shipping integration.
- **Non-functional**: high availability, correctness for money/inventory, gradual consistency elsewhere.

#### Domain decomposition (typical services)

- Catalog, Search, Cart, Order, Payment, Inventory, User, Shipping, Notification.

#### Consistency strategy

- Inventory:
  - Stronger consistency around “reserve/commit” (avoid oversell).
  - Use a reservation system with expiration and an order state machine.
- Use **sagas** (or workflow orchestrator) for checkout across services.
- Derived views: product search index, recommendations, analytics via async pipelines.

#### Hot paths

- Product page reads: cache + CDN.
- Search: dedicated search cluster.

---

### 15) Ride-sharing matching system (high level)

#### Requirements

- **Functional**: driver location updates, rider requests, matching, ETA, trip lifecycle.
- **Non-functional**: very low latency matching, correctness under rapid movement, high write throughput.

#### Core architecture

- Location ingestion service (high write QPS) → store in in-memory/KV with TTL.
- Geo indexing (geohash/S2 cells) to find nearby drivers quickly.
- Matching service:
  - fetch candidate drivers by geo cell + filters
  - compute ETA (coarse → refine)
  - dispatch offers; handle accept/decline/timeouts

#### Tradeoffs

- Consistency: location is inherently approximate; design for “good enough” with frequent updates.
- Hotspots: downtown cells become hot; split cells dynamically, use load-aware sharding.

---

## Cross-cutting “senior” topics you should weave into every design

### Reliability patterns

- **Timeouts everywhere** (per hop) and **bounded retries** (with jitter).
- **Circuit breakers** to stop cascading failures.
- **Backpressure**: queue limits, load shedding, concurrency caps.
- **Bulkheads**: isolate resources per tenant/endpoint.

### Data correctness patterns

- **Idempotency** for all writes that can be retried.
- **Outbox pattern** for reliable event publishing from DB-backed services.
- **Exactly-once is rare**; prefer at-least-once + dedupe.

### Observability

- **Golden signals**: latency, traffic, errors, saturation.
- Correlation IDs across services; structured logs; trace sampling.
- Alerts on symptoms (SLO burn rate) not just on raw metrics.

### Security & privacy

- Authn/authz, least privilege, secrets management.
- Encrypt in transit (TLS) and at rest.
- PII classification, data retention policies, audit logs.
- Abuse: rate limits, anomaly detection, spam/fraud defenses.

---

## Practice checklist (what to be able to answer quickly)

- **Scale**: estimate QPS, storage, bandwidth and identify top 2 bottlenecks.
- **Data**: pick a store and justify it; define partition keys and indexes.
- **Caching**: where, what, TTL/invalidation strategy; stampede protection.
- **Consistency**: what is strongly consistent vs eventually consistent and why.
- **Failures**: dependency down, partial outage, queue backlog, hot shard, data corruption.
- **Ops**: dashboards + alerts + runbooks; deploy strategy and rollback.

---

## Suggested study plan (4 weeks, adaptable)

- **Week 1**: fundamentals (one topic/day) + 2 mock designs (URL shortener, rate limiter)
- **Week 2**: chat + notifications + feed + caching deep dive
- **Week 3**: storage systems (S3-lite, distributed cache, video) + observability
- **Week 4**: payments + e-commerce + ride matching + 5 full mocks with timed practice
