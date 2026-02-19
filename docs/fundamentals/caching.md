# Caching (Redis concepts, eviction, invalidation) — simple but deep

Caching is one of the fastest ways to make systems feel “instant.”

But caching is also one of the easiest ways to create subtle bugs (stale data, weird inconsistencies, sudden outages from stampedes).

This guide explains caching in simple language, with the parts you need for system design interviews and real production systems.

---

## 1) What is a cache?

A **cache** is a fast place where you keep a copy of data so you don’t have to fetch it from a slower place every time.

Examples:

- Browser cache (images, CSS)
- CDN cache (static assets, public pages)
- Service cache (in-memory or Redis)
- Database buffer cache (inside the DB)

**Simple memory trick**:

- Database = “warehouse” (slower, reliable)
- Cache = “front desk” (fast, but not always up-to-date)

---

## 2) Why cache? (3 reasons)

### A) Lower latency

Cache hits avoid slow storage/network operations.

### B) Reduce load on the database

If you remove 90% of reads from the DB, the DB has more capacity for writes and complex queries.

### C) Handle traffic spikes

Caches can absorb bursts better than databases (especially when the DB is doing disk I/O).

---

## 3) What should be cached?

Cache things that are:

- read often
- expensive to compute
- slow to fetch
- okay to be slightly stale (or can be invalidated safely)

Good examples:

- user profile
- product page data
- permissions lookups (short TTL)
- feature flags (short TTL)
- “top posts today”

Be careful caching:

- money balances (unless you have strong correctness rules)
- inventory counts (risk of overselling)
- highly personalized data (cache key design is tricky)

---

## 4) Where to cache? (layers)

### A) Client-side (browser/mobile app)

- Fastest and cheapest.
- Risk: you must handle staleness and security (don’t cache private data publicly).

### B) CDN / edge cache

- Great for static and public content.
- Huge impact on latency globally.

### C) Service-side cache (Redis/memcache)

- Shared by many app instances.
- Typical for API responses, sessions, counters, etc.

### D) In-process cache (inside app memory)

- Fastest on a per-node basis.
- Risk: inconsistent across nodes; limited memory; cache “warms up” after restarts.

Senior tip: usually start with service-side cache (Redis) + CDN if applicable.

---

## 5) The 3 core caching patterns (easy to remember)

### Pattern 1: Cache-aside (lazy loading) — most common

How it works:

1. App checks cache.
2. If hit → return.
3. If miss → read DB, then write cache, then return.

Pros:

- simple
- cache only fills for hot data

Cons:

- first request after expiration is slow (cache miss)
- risk of **cache stampede** if many requests miss at once
- can serve stale data if invalidation is wrong

### Pattern 2: Write-through

How it works:

- On write, app writes to cache and DB together (synchronously).

Pros:

- cache stays fresh

Cons:

- slower writes
- cache becomes part of the write path (more failure scenarios)

### Pattern 3: Write-behind (write-back)

How it works:

- On write, update cache first.
- Later, background process flushes to DB.

Pros:

- very fast writes

Cons:

- data loss risk if cache dies before flush
- harder correctness story

Most interview designs use **cache-aside** unless there’s a strong reason otherwise.

---

## 6) Cache invalidation (the “hard part”)

There are only a few strategies, but you must pick one clearly.

### Strategy A: TTL-only (time-based expiration)

You set a TTL (time-to-live). After TTL, data is considered expired.

- Simple, reliable, and common.
- You accept some staleness until TTL expires.

Good for:

- feeds
- analytics
- configuration with periodic refresh

### Strategy B: Explicit invalidation on writes

When data changes in the DB, you update or delete the cache entry.

There are two common approaches:

- **Delete on write**: after DB update, delete cache key. Next read will refill.
- **Update on write**: after DB update, set the new value in cache.

Tradeoff:

- delete-on-write is safer (less chance of writing a wrong cached value),
- update-on-write can reduce misses but is easier to mess up.

### Strategy C: Versioned keys (simple and powerful)

Instead of deleting, you change the key:

- old key: `user:123:v7`
- new key: `user:123:v8`

Then you can keep old keys around until they expire naturally.

This is great when:

- deletes are expensive or unreliable,
- you want safe rollouts,
- you can store the “current version” somewhere reliable.

---

## 7) Cache stampede (dogpile) — what it is and how to stop it

### What is a stampede?

Many requests arrive at once, the cache is empty/expired, and all of them hit the DB together.

This can overload the DB and cause a full outage.

### Fix 1: Request coalescing (“single flight”)

Only one request is allowed to refresh the cache for a key. Others wait or serve stale.

Implementation ideas:

- distributed lock per key (careful with timeouts!)
- local single-flight in each instance + higher TTLs

### Fix 2: Soft TTL + background refresh

You keep two TTLs:

- **soft TTL**: after this, data is “stale but usable”
- **hard TTL**: after this, data must be refreshed

When soft TTL expires, refresh in background but still serve stale data briefly.

### Fix 3: Jitter your TTLs

If many keys expire at the same time (e.g., every hour), you create mini stampedes.

Add randomness:

- TTL = 1 hour ± 10%

### Fix 4: Pre-warm hot keys

For known hot items (home page, top posts), refresh before expiration.

---

## 8) Eviction (what happens when cache runs out of memory)

Caches have limited memory. When full, they must remove (“evict”) items.

Common policies:

- **LRU** (Least Recently Used): remove items not used recently.
- **LFU** (Least Frequently Used): remove items used least often.
- **Random**: simple, sometimes good enough.

What to remember:

- LRU works well for many workloads.
- LFU is great when there are strong “hot items,” but can be more complex.

Senior tip: always mention **hot keys** — one hot key can dominate memory and CPU.

---

## 9) Redis basics (only what you need)

Redis is often used as:

- a distributed cache
- a session store
- a counter store
- a lightweight message stream (sometimes)

### Data structures you should know

- **String**: simplest value (good for JSON blobs)
- **Hash**: map/dictionary (good for per-user fields)
- **Set**: unique members
- **Sorted set**: scoring/ranking (leaderboards, timelines)
- **Stream**: append-only stream (event processing)

### Atomic operations (important for correctness)

Redis can do atomic updates:

- `INCR` for counters
- `SETNX` for “set if not exists” locks (with TTL!)
- Lua scripts for multi-step atomic logic (token bucket rate limiting)

### Persistence/availability (don’t overcomplicate)

For caching, many systems accept:

- Redis may lose some data (because it’s a cache), but should be highly available.

If Redis is a source of truth (sessions, rate limits), discuss:

- replication
- failover
- how you handle data loss (often “acceptable” for ephemeral state)

---

## 10) Common caching bugs (and how to avoid them)

### Bug A: caching the wrong thing (missing dimensions)

Example: caching a response without including `user_id` in the cache key.

Fix:

- design cache keys carefully (include locale, auth state, user segments, etc.)

### Bug B: cache poisoning / security issues

Fix:

- never cache private data in shared caches unless key includes auth/tenant identity
- use proper cache-control headers at CDN

### Bug C: stale data surprises

Fix:

- pick one invalidation strategy and say it clearly
- start with TTLs if you can tolerate staleness
- for strict correctness, avoid caching that data or use explicit invalidation with care

### Bug D: “cache is down” becomes “DB is down”

If cache fails, everyone hits DB at once.

Fix:

- graceful degradation (serve stale for non-critical paths)
- rate limit and shed load
- circuit breakers to protect DB

---

## 11) Interview-ready summary (simple talk track)

“We’ll use cache-aside with Redis to reduce DB reads and latency. Cache keys include user/locale where needed. We’ll use TTLs plus explicit invalidation for critical entities. To avoid stampedes we’ll add TTL jitter and single-flight refresh. Under cache failure we’ll degrade by serving stale data when acceptable and protect the DB with rate limits and circuit breakers.”

