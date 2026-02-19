# System design interview template (copy/paste) + how to use it

This is a *battle-tested* structure for system design interviews. It’s intentionally repetitive: the goal is to keep you coherent under time pressure and make tradeoffs explicit.

Use it in two ways:

- **In interviews**: as your live outline on the whiteboard.
- **In prep**: as the format for your own 1–2 page “design docs” per practice problem.

---

## The template (copy/paste)

Copy this into your notes and fill it in live.

### 1) Requirements

**Functional**

- Primary user flows (top 2–3)
- Secondary flows (nice-to-have)
- Edge cases (idempotency, duplicates, ordering)

**Non-functional (pick 2–4 that truly matter)**

- Latency targets (p50/p95/p99)
- Availability target (e.g., 99.9% / 99.99%)
- Durability (data loss tolerance)
- Consistency requirements (strong vs eventual; which entities?)
- Throughput and growth expectations
- Security/compliance (PII, GDPR, PCI, etc.)
- Cost constraints

**Out of scope**

- Explicitly list what you will not build.

**Acceptance criteria**

- 3–5 bullets that define “this is correct and done.”

---

### 2) Scale estimate (back-of-the-envelope)

- Users (DAU/MAU), geo distribution
- QPS (avg + peak factor), read/write ratio
- Payload sizes → bandwidth
- Storage growth and retention
- Hot key / hot partition risk

Outcome: identify the **top 1–2 bottlenecks** that will shape your design.

---

### 3) API design

For each core operation:

- Endpoint + request/response shape
- Pagination strategy (cursor preferred)
- Error codes and semantics
- **Idempotency keys** for writes
- Authn/authz model (who can call it?)

---

### 4) Data model

- Entities and relationships
- Primary keys, secondary indexes
- Access patterns (top queries)
- Partition/shard key (if needed)
- Retention / TTL policies

---

### 5) High-level architecture

- Main components and responsibilities
- Data flow for read path and write path
- Sync vs async boundaries
- Caching layers (if any)
- Multi-region considerations (if in scope)

---

### 6) Deep dives (pick 2–3)

Examples:

- Caching + invalidation strategy + stampede protection
- Partitioning/sharding + resharding plan
- Consistency model and UX guarantees
- Queue semantics (at-least-once, dedupe, DLQ)
- Search/indexing pipeline (derived views)
- Overload behavior (rate limits, backpressure, load shedding)

---

### 7) Reliability / failure modes

- Dependency failures and fallbacks
- Timeouts + retries + circuit breakers
- Data corruption and recovery (backups, replays)
- Disaster recovery (RPO/RTO)

---

### 8) Observability & operations

- Golden signals metrics and dashboards
- Structured logging + tracing + correlation IDs
- Alerting strategy (symptom-based, SLO burn rate)
- Deployment safety (canary, feature flags, rollbacks)
- Migration/backfill strategy

---

### 9) Security & abuse

- Authn/authz, secrets management
- Encryption in transit/at rest
- PII handling and retention
- Abuse prevention (rate limits, spam/fraud signals)

---

### 10) Tradeoffs and next steps

- 2–3 key tradeoffs you made, and why
- Evolution plan when scale grows 10×
- “If I had more time…” (nice-to-haves)

---

## How to use the template in a real interview (timeboxing)

A common 35–45 minute loop:

- **0–5 min**: requirements + out-of-scope
- **5–10 min**: scale estimate
- **10–18 min**: high-level architecture
- **18–35 min**: deep dives + failure modes
- **35–45 min**: ops/security + wrap-up + tradeoffs

If the interview is 60 minutes, you can afford an extra deep dive (data model + partitioning is usually the best choice).

---

## What “good” looks like at each stage

### Requirements: crisp scope, crisp guarantees

High-signal phrasing:

- “We’ll support X and Y flows; we won’t build Z today.”
- “Reads should be p95 < 150ms; writes can be p95 < 1s.”
- “Inventory must be strongly consistent; the product feed can be eventual.”

### Scale estimate: numbers that change the design

High-signal phrasing:

- “Redirect path is 99% reads at peak 200k QPS; this forces edge caching + hot key strategy.”
- “Writes are small but frequent; we need a log/queue to absorb spikes and keep DB stable.”

### API: idempotency and pagination are senior signals

Always call out:

- “All mutating endpoints accept an idempotency key.”
- “We use cursor pagination to avoid inconsistent page boundaries under concurrent writes.”

### Data model: access patterns first

High-signal phrasing:

- “Our hot query is ‘fetch latest 50 items by user’; we model it to be a single partition read.”
- “Indexes: we index by X because that’s the filter; we include columns Y,Z as a covering index for the list view.”

### Architecture: minimal components, clear responsibilities

High-signal phrasing:

- “This boundary is asynchronous because it’s not user-visible latency and it isolates failures.”
- “We keep the serving tier stateless; state lives in durable stores.”

---

## Deep dive selection: how to choose quickly

Pick deep dives based on the bottleneck:

- **Read-heavy, latency-critical**: caching + invalidation, edge/CDN, data locality
- **Write-heavy**: partitioning, batching, queue semantics, backpressure
- **Multi-region**: replication, consistency model, conflict resolution
- **Abuse-prone**: rate limiting, quotas, fraud/spam defenses

Pick deep dives based on risk:

- Money/inventory → correctness + idempotency + state machines
- Realtime chat/streaming → ordering, fanout, connection management
- Search/feed → derived views, indexing pipelines, eventual consistency

---

## A worked mini-example (so you see the template in action)

Example prompt: “Design a URL shortener.”

### Requirements (excerpt)

- Functional: create short link, redirect; optional custom alias and expiration
- Non-functional: redirect p99 < 50ms, high availability; eventual consistency ok for analytics
- Out of scope: user accounts, advanced anti-abuse, multi-tenant enterprise features

### Scale estimate (excerpt)

- 100M redirects/day → ~1,157 QPS average, 10× peak → ~11.5k QPS peak
- Redirect payload is small; bandwidth dominated by TLS and round trips; p99 latency drives edge caching

### Deep dives you’d likely choose

- Hot key caching at edge + cache stampede protection
- Code generation strategy + collision handling
- Analytics pipeline via async events

This is exactly how the template keeps you focused: it pushes you toward the few decisions that matter.

---

## Common template mistakes (and fixes)

- **Mistake**: writing every possible non-functional requirement.  
  **Fix**: choose 2–4 that materially change architecture.

- **Mistake**: APIs with no idempotency or pagination plan.  
  **Fix**: always state idempotency keys for writes; cursor pagination for list endpoints.

- **Mistake**: “We’ll use Kafka” without semantics.  
  **Fix**: specify at-least-once delivery, dedupe strategy, retries, DLQ.

- **Mistake**: “We’ll shard the DB” with no key.  
  **Fix**: state partition key, hotspot risks, resharding approach, and cross-shard query story.

