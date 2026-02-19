# Rate limiting — simple, deep guide

Rate limiting means **putting a speed limit on requests** so your system stays healthy and fair.

It’s used to:

- stop abuse (bots, brute force, scraping)
- protect expensive endpoints (search, checkout)
- prevent cascading failures during spikes
- enforce quotas (per user, per API key, per IP)

If you remember one line:

> Rate limiting is how you keep the system usable when traffic is not “nice.”

---

## 1) What rate limiting looks like in practice

Examples:

- “Max 100 requests per minute per user”
- “Max 10 login attempts per minute per IP”
- “Max 5 checkout requests per second per user”
- “Max 1,000 requests per day per API key”

Rate limiting is usually enforced at the **edge** (CDN / API gateway) because that’s the cheapest place to stop traffic.

---

## 2) Where to enforce rate limits (layers)

- **CDN/edge**: best for blocking obvious abuse early.
- **API gateway**: common for per-key/per-user limits and consistent enforcement.
- **Service level**: useful for protecting internal dependencies (DB, downstream services).
- **Per-resource**: protect a specific heavy feature (e.g., “export reports”).

Senior tip: use more than one layer when needed (defense in depth).

---

## 3) The 4 common algorithms (easy to remember)

### A) Fixed window

“Count requests in a time bucket (like per minute).”

- Simple.
- Problem: bursts at window boundaries (59s and 60s can double).

### B) Sliding window (log or counter)

“Count requests in the last N seconds, moving continuously.”

- More accurate and fair.
- More state/complexity than fixed window.

### C) Token bucket (most common for APIs)

Imagine a bucket of tokens:

- tokens refill at a steady rate
- each request spends 1 token
- if bucket is empty → reject (429)

Why it’s great:

- allows **bursts** (if the bucket has saved tokens)
- keeps average rate bounded

### D) Leaky bucket

Think “requests enter a queue and leak out at a steady rate.”

Why it’s useful:

- smooths traffic
- protects downstream systems by controlling processing rate

---

## 4) What to return when limiting

Common response:

- HTTP **429 Too Many Requests**

Helpful headers:

- `Retry-After`
- optional: remaining quota, reset time

Beginner-friendly best practice:

- be clear to clients *when they can try again*.

---

## 5) Distributed rate limiting (the real challenge)

If you have many API servers, each server can’t rate limit independently — otherwise a user can “spread” requests across servers.

You need shared state.

### Common approach: Redis

Store counters/tokens in Redis:

- key: `rl:user:123:route:/search`
- value: counter or token state
- TTL: window duration

Atomicity matters:

- use atomic operations (INCR + TTL) or a Lua script for token bucket logic

### Tradeoff: correctness vs latency

Central Redis checks add latency and can become a bottleneck.

Two common strategies:

- **Central exact-ish limit** (Redis): more accurate, a bit slower.
- **Local limit + periodic sync**: faster, slightly approximate.

In interviews, it’s fine to say:

- “We’ll do Redis-based limiting; if Redis is down we fall back to a local limiter to keep availability.”

---

## 6) Key design: what do you limit by?

You can limit by:

- IP address (good for unauthenticated traffic; not perfect due to NAT)
- user ID (best when authenticated)
- API key / tenant ID (common for B2B)
- route/endpoint (protect expensive endpoints)
- combination: `(user_id, endpoint)`

Be careful:

- IP-based can punish many users behind one NAT.
- user-based requires auth; attackers can create accounts.

Often you use multiple:

- IP limit for “unauthenticated”
- user limit for “authenticated”

---

## 7) Overload behavior: rate limiting vs load shedding

- **Rate limiting**: “You’re over your quota/fair share.”
- **Load shedding**: “System is overloaded; we must drop work to survive.”

Both can return 429/503 depending on policy.

Senior signal: mention you’ll protect the DB with:

- concurrency limits
- queue length limits
- circuit breakers

---

## 8) Interview-ready summary

“We enforce rate limits at the edge/gateway to protect the system and ensure fairness. We use token bucket for bursty APIs and return 429 with `Retry-After`. In a distributed setup we keep shared counters/tokens in Redis with atomic operations (Lua/INCR+TTL). We limit by user/API key and endpoint, and under Redis failure we fall back to a local approximate limiter to preserve availability.”

