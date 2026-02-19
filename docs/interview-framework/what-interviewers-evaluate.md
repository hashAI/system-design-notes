# What interviewers evaluate (and how to show it)

System design interviews are not about reciting components. They’re about demonstrating senior judgment under constraints: what you choose, what you *don’t* choose, and how you reason about tradeoffs and failure.

This page explains the common evaluation rubric used in product companies, how to “signal” each dimension in your design, and how to avoid the most frequent failure modes.

---

## 1) Requirements clarity (problem framing)

### What they’re looking for

- You can **turn a vague prompt into a concrete problem**.
- You can **separate must-haves from nice-to-haves**.
- You pick a scope that is **deliverable within interview time**.
- You state and validate assumptions quickly.

### How to demonstrate it

- Start with 30–90 seconds of questions that establish:
  - **Primary user flows** (who, what, when, how often).
  - **Read/write paths** (latency-critical vs async acceptable).
  - **Core constraints** (region, privacy, cost, compliance).
  - **Out of scope** to prevent “boil the ocean.”
- Repeat back the problem as a crisp statement:
  - “We’re building X for Y users; we must support A/B/C with p95 latency ≤ N ms; we can accept eventual consistency for D, but not for E.”

### Common pitfalls

- Jumping straight to microservices before you’ve defined the product.
- Treating every feature request as a requirement.
- Not stating what you are *not* building.

### Interviewer “tells”

If the interviewer keeps asking “so what are we building exactly?” you’re failing this dimension. Fix by: summarizing requirements + explicit out-of-scope + success metrics.

---

## 2) System correctness (semantics and invariants)

### What they’re looking for

- You understand the **core invariants** that must hold true.
- You can define **edge cases** (retries, duplicates, concurrency).
- You can reason about **state machines** and **idempotency**.

### How to demonstrate it

For each top-level flow, call out:

- **Uniqueness** constraints (e.g., “each short code maps to one URL”).
- **Ordering** constraints (e.g., “chat messages are ordered per conversation”).
- **Exactly-once vs at-least-once** expectations (and how you handle duplicates).
- **Idempotency keys** for write APIs that can be retried.
- **Concurrency control** choices:
  - optimistic concurrency (version fields, compare-and-swap)
  - pessimistic locks (only when necessary)
  - transactional boundaries (what must be atomic)

### Quick “correctness checklist” you can apply live

- **Retry-safe writes**: which APIs are idempotent?
- **Duplicate events**: can consumers safely process an event twice?
- **Lost updates**: what happens if two clients update the same record?
- **Partial failure**: what happens if step 2 succeeds and step 3 fails?
- **Time**: are you relying on wall-clock time (clock skew) or monotonic sequences?

### Common pitfalls

- “We’ll use Kafka for reliability” without specifying delivery semantics and dedupe.
- “We’ll store it in Redis” without describing durability and recovery.

---

## 3) Scalability (growth path, bottlenecks, and “10× thinking”)

### What they’re looking for

- You can estimate scale to pick an architecture that will hold.
- You can identify bottlenecks and propose a credible scaling path.
- You know when “add servers” stops working and **data partitioning** becomes required.

### How to demonstrate it

Do quick back-of-the-envelope math:

- Traffic: DAU/MAU, QPS, read/write ratio, peak factor
- Bandwidth: response sizes × QPS
- Storage growth: writes/day × bytes/write × retention
- “What dominates cost?” (CPU, network egress, storage, cache)

Then anchor your design to those numbers:

- Stateless service tier behind load balancer for horizontal scaling
- Cache placement for latency and load reduction
- Database scaling plan:
  - indexes
  - read replicas
  - partitioning/sharding (partition key + reshard strategy)

### Bottleneck articulation (what good sounds like)

Instead of “we can scale by adding nodes,” say:

- “Redirect traffic is read-heavy; the bottleneck is DB read IOPS at p99. We’ll use cache-aside with TTL + stampede protection. Hot keys can be edge cached. The DB becomes a write-oriented source of truth.”

### Common pitfalls

- Capacity numbers that don’t influence decisions.
- “Sharding” as a magic word without a partition key and operational story.

---

## 4) Reliability & resilience (degraded modes, overload, and recovery)

### What they’re looking for

- You anticipate failures and design graceful degradation.
- You know standard resilience patterns and when to apply them.
- You avoid creating cascading failures.

### How to demonstrate it

Pick 3–5 likely failures and walk through behavior:

- Dependency down (cache / DB / queue / third-party)
- Partial outage (one AZ down, one region impaired)
- Queue backlog / consumer lag
- Hot partition / hot key
- Data corruption / bad deploy

Then propose mitigations:

- **Timeouts** with per-hop budgets
- **Retries** (bounded, exponential backoff, jitter) and only for idempotent operations
- **Circuit breakers** to prevent retry storms
- **Bulkheads** (isolate per tenant / per endpoint)
- **Backpressure**: queue length limits, concurrency caps
- **Load shedding**: 429/503 with clear semantics
- **Failover**: active-active vs active-passive
- **Disaster recovery**: backups, restore drills, RPO/RTO targets

### Degradation design (a high-signal move)

Explicitly define what happens under overload:

- Serve stale cache for non-critical reads
- Disable expensive features (“brownout”)
- Reduce write amplification (buffering, sampling, batching)
- Apply priority to critical endpoints (payments > analytics)

---

## 5) Data thinking (modeling, access patterns, indexes, partitioning)

### What they’re looking for

- You model entities correctly for your access patterns.
- You can specify indexes, keys, and queries.
- You can reason about partitions and cross-partition operations.

### How to demonstrate it

For each core entity, provide:

- Primary key and key shape (UUID, Snowflake, composite key)
- Secondary indexes (and why)
- Most frequent queries and their complexity
- Partition key choice and “hotspot” analysis

Explain tradeoffs in store selection:

- Relational when invariants and joins matter
- Key-value for simple access patterns and scale
- Wide-column for write-heavy, query-designed schemas
- Search index as a derived view for full-text and aggregations

### Common pitfalls

- “We’ll use NoSQL for scalability” without the query model.
- Ignoring migration and schema evolution.

---

## 6) Tradeoffs & decision-making (senior judgment)

### What they’re looking for

- You can articulate alternatives and choose one based on constraints.
- You avoid over-engineering while still planning for growth.

### How to demonstrate it

Use the pattern:

- **Option A** (why it’s tempting)
- **Option B** (what it buys, what it costs)
- **Decision**: “Given constraints X and Y, we choose B because…”
- **Growth path**: “If we hit bottleneck Z, we evolve to…”

Examples of high-signal tradeoffs:

- Consistency vs latency in multi-region reads
- Fanout-on-write vs fanout-on-read for feeds
- Replication vs erasure coding for storage durability and cost
- Centralized vs local rate limiting correctness and availability

---

## 7) Communication (structure, clarity, and leadership)

### What they’re looking for

- You drive the conversation, keep it structured, and make it easy to follow.
- You narrate the design at the right level and dive deeper on request.

### How to demonstrate it

- Use a consistent structure:
  - requirements → scale estimate → APIs → data model → high-level architecture → deep dives → failures/ops → tradeoffs
- Every time you add a component, say:
  - “Why it exists” (goal)
  - “How it scales”
  - “How it fails”

### Common pitfalls

- Whiteboarding a “component soup.”
- Getting stuck in implementation details (e.g., schema minutiae) too early.

---

## 8) Operability (metrics, debugging, deployments)

### What they’re looking for

- You can run the system in production.
- You know what to measure and how to respond to incidents.

### How to demonstrate it

Mention:

- **Golden signals**: latency, traffic, errors, saturation
- **SLIs/SLOs**: what “good” means and what triggers an alert
- **Tracing**: correlation IDs across services
- **Dashboards**: per dependency, per endpoint, per region
- **Deploy safety**: canary, feature flags, rollbacks
- **Data migrations**: dual writes/reads, backfills, verification

---

## 9) Security & abuse (auth, privacy, and adversaries)

### What they’re looking for

- You don’t treat security as an afterthought.
- You understand common abuse vectors for the product.

### How to demonstrate it

- Authn/authz model: users/roles/tenants, token strategy (JWT vs opaque)
- Data protection: TLS, encryption at rest, secrets management
- PII handling: classification, retention, audit logs
- Abuse: rate limiting, anomaly detection, spam/fraud checks

---

## A practical “scoring rubric” (mental model)

Many interviewers effectively score something like:

- **Pass**: clear requirements, coherent architecture, basic scaling + failure modes, correct tradeoffs.
- **Strong pass**: accurate data model and partitioning/caching choices; excellent failure/degradation thinking; crisp communication; strong operability/security instincts.
- **No hire**: unclear scope; over/under engineered; can’t articulate tradeoffs; ignores correctness and failures.

---

## Recommended talk track (5–10 minutes to set up success)

Use this as your opening:

1. Confirm requirements and define out-of-scope.
2. Do a fast scale estimate (QPS, storage, bandwidth) and highlight likely bottlenecks.
3. Present a high-level architecture with the main components and the sync/async boundaries.
4. Pick 2 deep dives based on the bottleneck and the product:
   - caching + partitioning
   - consistency model and UX guarantees
   - reliability patterns for overload/failure
5. Close with operations, security/abuse, and a growth path.

