# Design a Rate Limiter (as a service / gateway) — full system design

Rate limiting is a “protect the whole system” component. It appears in API gateways, CDNs, service meshes, and even inside individual services.

This design focuses on a **distributed rate limiter** that can be used:

- at an API gateway (north-south traffic)
- as a shared internal service (east-west traffic)

---

## 1) Requirements

### Functional requirements

- Support limits by:
  - user ID (authenticated)
  - API key / tenant ID
  - IP address (unauthenticated)
  - endpoint/route (e.g., `/login`, `/search`)
  - combinations like `(tenant_id, route)` or `(user_id, route)`
- Support common policies:
  - requests per second/minute/day
  - burst handling (allow short bursts)
  - concurrency limits for expensive endpoints (optional)
- Return correct errors:
  - HTTP `429 Too Many Requests`
  - include `Retry-After` and/or remaining quota headers
- Configurable policies that can be updated without redeploying data plane (control plane vs data plane)

### Non-functional requirements

- **Low latency overhead**: limiter should add only a few ms.
- **High availability**: limiter should not become a single point of failure.
- **Good-enough correctness**: exact correctness is nice, but systems usually accept small inaccuracies for availability/latency.
- **Fairness**: prevent one tenant/user from starving others.
- **Scalability**: handle high QPS at the edge.

### Out of scope (unless asked)

- Full billing/entitlement systems
- Complex dynamic pricing/credits
- ML-based abuse detection (can be a downstream consumer of rate limit signals)

---

## 2) Back-of-the-envelope scale

Assume:

- gateway peak traffic: 200k requests/sec
- rate limiting check for every request

Implications:

- If you do a remote store check for every request, that store must handle ~200k ops/sec (plus bursts).
- You need:
  - very fast storage (Redis cluster / in-memory with sync)
  - careful key design to avoid hotspots
  - graceful degradation/fallback behavior

---

## 3) API / behavior

Rate limiting is often not exposed as a public API; it’s an internal decision:

Input to limiter:

- identity (user_id/api_key/ip)
- route (normalized path template like `/v1/orders/{id}` rather than raw URL)
- method (GET/POST)
- timestamp (for windows)

Output:

- allow/deny
- remaining quota
- retry-after

Example HTTP response headers (common pattern):

- `X-RateLimit-Limit: 100`
- `X-RateLimit-Remaining: 7`
- `X-RateLimit-Reset: 1700000123`
- `Retry-After: 3`

---

## 4) Policies and algorithms (what to implement)

### A) Token bucket (best default for APIs)

Mental model:

- bucket has capacity `B` tokens
- tokens refill at rate `r` tokens/sec
- each request consumes 1 token

Why it’s good:

- enforces an average rate
- allows bursts up to `B`

Example:

- “10 req/sec with bursts up to 50”
  - `r = 10`, `B = 50`

### B) Fixed window (simple, can be bursty)

- count requests in a window like “per minute”
- easy to implement with `INCR` + TTL

Problem:

- boundary bursts: you can send 100 at 12:00:59 and 100 at 12:01:00

### C) Sliding window (fairer, more state)

More accurate than fixed window, but requires more tracking.

### D) Concurrency limiting (protect expensive endpoints)

Different from rate per time:

- “only 20 in-flight requests for `/export` per tenant”

Often implemented with:

- semaphore-like counters with TTL/leases

---

## 5) Architecture (data plane + control plane)

### Data plane (fast path)

Lives in:

- API gateway
- sidecar proxy
- edge service

Responsibilities:

- evaluate rate limit decision quickly
- apply local caching to reduce remote store calls
- return 429 when denied

### Control plane (slow path)

Responsibilities:

- manage policies (CRUD)
- validate policy rules
- distribute configs to data plane (push) or store in config service (pull)
- audit changes and rollbacks

Why separate them:

- data plane must be fast and stable
- control plane changes frequently

---

## 6) Storage choices for distributed enforcement

### Option 1: Central store (Redis) — common and interview-friendly

Approach:

- each request updates a counter/token state in Redis
- use atomic ops or Lua scripts to avoid race conditions

Pros:

- globally consistent-ish limit
- simple to reason about

Cons:

- Redis becomes a hot dependency
- added network hop and tail latency

### Option 2: Local limiter + periodic sync (approximate)

Approach:

- each gateway instance enforces locally
- periodically syncs usage (or uses leased “token blocks”)

Pros:

- very low latency
- survives central store failures better

Cons:

- less accurate
- may allow more than quota during bursts or failures

Practical hybrid:

- use Redis for “hard” quota limits (per minute/hour/day)
- use local limiter for “soft” per-second smoothing

---

## 7) Data model (Redis keys and avoiding hotspots)

### Key design

Common key formats:

- `rl:{tenant_id}:{route}:{window_start}` (fixed window)
- `tb:{tenant_id}:{route}` (token bucket state)
- `conc:{tenant_id}:{route}` (concurrency)

Normalize route:

- use route templates to avoid exploding key cardinality:
  - good: `/v1/orders/{id}`
  - bad: `/v1/orders/9f2a...`

### Avoiding hotspots

Hot keys happen for:

- big tenants
- login endpoints during attacks

Mitigations:

- shard counters by adding a small suffix:
  - `rl:{tenant}:{route}:{window}:{shard}`
  - then sum across shards (tradeoff: extra reads)
- separate endpoints into different Redis clusters
- add WAF/CDN limits before gateway limits

---

## 8) Token bucket with Redis (how it works, conceptually)

Token bucket state per key:

- `tokens` (float/int)
- `last_refill_ts`

On each request:

1. compute how many tokens to add since last refill
2. cap at bucket size
3. if tokens >= 1 → decrement and allow
4. else → deny and compute retry-after

Must be atomic:

- use a Lua script so read/compute/write is one operation

This is a very common “senior” detail to mention.

---

## 9) Failure modes and fallbacks (important!)

### Redis is down / slow

You must choose a policy:

- **Fail-open** (allow traffic):
  - better availability
  - risk: backend overload during attacks
- **Fail-closed** (deny traffic):
  - protects backend
  - risk: legitimate users get blocked

Common approach:

- fail-open for low-risk endpoints
- fail-closed for high-risk endpoints (login, expensive operations)
- always protect critical downstream with additional circuit breakers/load shedding

### Network partitions / partial outages

Rate limiting is often “best effort.”

State your guarantee clearly:

- “We guarantee limits approximately; we prioritize availability and backend protection.”

### Clock skew

Window-based algorithms depend on time.

Mitigation:

- prefer token bucket state using elapsed time in the store script
- if using fixed windows, compute window boundaries consistently (server-side)

---

## 10) Observability (how you know it’s working)

Metrics:

- allowed vs blocked requests (by route/tenant)
- Redis latency and error rate
- policy evaluation errors (bad config)
- top limited keys (identify abuse)
- fallback mode activations (fail-open/closed)

Dashboards:

- blocked rate spikes → indicates attacks or misconfiguration

---

## 11) Example policies (concrete and memorable)

### Example 1: Public API

- per API key:
  - 10 req/sec burst 50 (token bucket)
  - 100k req/day hard cap (fixed window day)

### Example 2: Login endpoint

- per IP: 5/min (fixed/sliding window)
- per username: 10/hour (to slow brute force)
- fail-closed if limiter store is down (security-first)

### Example 3: Expensive export job

- per tenant: 2 concurrent exports
- plus: 1 export/minute enqueue limit

---

## 12) Interview-ready summary (30 seconds)

“We enforce rate limits at the edge/gateway with token bucket for bursty traffic and optional fixed-window caps for daily quotas. The data plane evaluates decisions fast, using Redis for shared atomic counters/token state (Lua scripts). We normalize keys by `(tenant/user, route)` to keep cardinality manageable, handle hotspots with sharding and multi-layer limits (CDN/WAF first), and define clear fallback behavior (fail-open vs fail-closed) when Redis is unavailable. We monitor allow/deny rates, Redis latency, and top limited keys.”

