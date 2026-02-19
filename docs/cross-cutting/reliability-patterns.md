# Reliability patterns — what to weave into every system design

Reliability is how systems behave under:

- partial failures
- slow dependencies
- overload
- bad deployments

This page is a set of practical patterns you can apply to almost any design.

---

## 1) Timeouts everywhere (the most important rule)

If you don’t set timeouts, your system can hang until threads and connections are exhausted.

Where to set timeouts:

- client → gateway
- gateway → service
- service → DB/cache/queue/third party

Good practice:

- set per-hop timeouts, not one giant timeout
- keep timeouts tight enough to fail fast, but not so tight you cause unnecessary retries

Simple guideline:

- total request budget = p99 target
- split budget across hops, leaving room for retries

---

## 2) Retries (bounded, jittered, and only when safe)

Retries are useful for transient failures but can easily create outages.

### When to retry

- idempotent reads (GET)
- idempotent writes (POST with idempotency key)
- transient errors (timeouts, connection resets, some 5xx)

### When NOT to retry

- non-idempotent writes without idempotency keys
- when backend is overloaded (you’ll amplify the overload)

### Best practices

- **cap retries** (often 1 retry is enough)
- exponential backoff + **jitter**
- per-try timeout (don’t just extend total time)

---

## 3) Circuit breakers (stop hammering a failing dependency)

If a dependency is failing, stop sending traffic for a short period.

States:

- closed: normal
- open: fail fast
- half-open: probe recovery

Benefits:

- prevents retry storms
- reduces tail latency
- gives dependencies room to recover

---

## 4) Backpressure (don’t accept infinite work)

Systems fail when they accept more work than they can process.

Backpressure tools:

- bounded queues
- concurrency limits
- request admission control

Example:

- if queue depth > threshold, reject new work or degrade.

---

## 5) Load shedding (graceful failure under overload)

When overloaded, you must drop some work intentionally.

Techniques:

- return 429/503 early at gateway
- disable expensive features (“brownout”)
- serve cached/stale data for non-critical reads
- prioritize critical endpoints (payments > analytics)

The goal is:

- protect the core system so it can recover

---

## 6) Bulkheads (isolation)

Bulkheads prevent one noisy part from taking down the whole ship.

Examples:

- separate thread pools per endpoint
- separate DB connection pools per workload
- separate queues per tenant or priority

---

## 7) Idempotency (the foundation of safe retries)

Any operation that might be retried should be idempotent.

Common approach:

- client sends `idempotency_key`
- server stores result keyed by (client, idempotency_key)
- retries return the same result

This is essential for:

- payments
- order creation
- notification sends

---

## 8) Rate limiting and quotas (protect from abuse and accidents)

Rate limiting protects:

- infrastructure from spikes
- expensive endpoints from abuse
- fairness between tenants

Apply at multiple layers:

- CDN/WAF
- gateway
- service

---

## 9) Graceful degradation (define what happens when X is down)

A senior design always answers:

- “If cache is down, what happens?”
- “If DB is down, what still works?”
- “If queue is backed up, what do users see?”

Examples:

- serve stale feed if ranking service is down
- disable analytics writes when queue is overloaded
- allow reads but block writes during partial outage (or vice versa)

---

## 10) Deployment safety

Most outages are caused by changes.

Practices:

- canary releases
- feature flags
- dark launches / shadow traffic
- fast rollback
- schema migrations with backward compatibility

---

## 11) Disaster recovery basics (RPO/RTO)

Know these terms:

- **RPO**: how much data loss is acceptable (time)
- **RTO**: how long downtime is acceptable

Practices:

- backups and restore drills (not just “we have backups”)
- multi-AZ replication
- multi-region failover for critical systems

---

## 12) Interview-ready summary

“I design for reliability by setting timeouts everywhere, using bounded retries with jitter only for idempotent operations, and adding circuit breakers to prevent cascading failures. I enforce backpressure with bounded queues and concurrency limits, and shed load gracefully under overload by degrading non-critical features. I isolate components with bulkheads, protect APIs with rate limits, and deploy safely with canaries and feature flags. For critical systems, I define RPO/RTO and ensure backups and failover are tested.”

