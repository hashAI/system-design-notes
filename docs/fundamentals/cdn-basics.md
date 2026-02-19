# CDN basics — practical deep dive

A **CDN (Content Delivery Network)** is a network of servers around the world that helps deliver content faster and cheaper.

A CDN caches and serves content from edge POPs to reduce latency and origin load. It also commonly provides TLS termination and DDoS/WAF capabilities.

---

## 0) The default CDN plan (works for most products)

- Put all **static assets** (images/CSS/JS/video segments) behind a CDN.
- Use **versioned URLs** for assets so deploys don’t require mass invalidation.
- Keep **HTML** TTL short; keep **assets** TTL long.
- Be explicit about **cache keys** (avoid caching personalized/private responses incorrectly).
- Monitor **cache hit ratio** and **origin load**.

---

## 1) The “origin vs edge” picture

- **Origin**: your main backend/server (where content “really” lives).
- **Edge / POP**: CDN servers near users (Points of Presence).

When a user requests an image:

1. User hits a nearby edge server.
2. If edge has it cached → return immediately (**cache hit**).
3. If not → edge fetches from origin, stores it, then returns (**cache miss**).

Key concepts:

- cache keys and variation (query params, headers)
- TTL and cache-control
- invalidation vs versioned URLs
- private content (signed URLs/cookies)

---

## 2) What CDNs are used for

### A) Static assets (most common)

- images
- CSS/JS
- videos segments
- downloadable files

### B) Cacheable API responses (sometimes)

Examples:

- public endpoints
- “top posts” for anonymous users

But you must be careful with cache keys so you don’t leak private data.

### C) Performance and security at the edge

Many CDNs also provide:

- TLS termination
- WAF (web application firewall)
- DDoS protection
- bot management
- rate limiting

---

## 3) Cache keys (how the CDN decides “same content”)

A CDN decides caching based on a **cache key**.

By default, a cache key might include:

- URL path
- query parameters (sometimes)
- request headers (sometimes)

This is a very common bug source.

### A classic bug

If you serve personalized content but your cache key doesn’t include user identity:

- user A’s content can be served to user B.

Fix:

- don’t cache private responses at CDN, or
- include the right headers/cookies in the cache key, or
- use signed URLs for protected content.

---

## 4) TTL and cache-control (who decides how long to cache)

The origin usually tells the CDN how long to cache using HTTP headers like:

- `Cache-Control: max-age=...`
- `Cache-Control: s-maxage=...` (shared caches)

Simple idea:

- small TTL = fresher but more origin load
- large TTL = faster and cheaper but staler

---

## 5) Invalidation (how to update cached content)

Sometimes content changes and you need the CDN to stop serving the old version.

Options:

### A) Wait for TTL

Let it expire naturally.

### B) Purge/invalidate

Tell the CDN: “remove this path from cache.”

Tradeoff:

- invalidation is sometimes slow/expensive
- invalidating many objects can be costly

### C) Versioned URLs (best practice for assets)

Instead of invalidating, change the URL:

- `/app.js` → `/app.v123.js`

Now new deploys automatically use the new file. Old cached versions can expire naturally.

This is how most modern frontend asset pipelines work.

---

## 6) Range requests (important for large files and video)

For video and large files, clients often request only parts:

- “give me bytes 1,000,000 to 2,000,000”

CDNs support **range requests** so:

- playback starts quickly
- users can seek without downloading everything

This is a key piece of video streaming design (HLS/DASH segment delivery).

---

## 7) Signed URLs / cookies (private content)

If content should be private (e.g., paid video), you can use:

- **signed URLs** (URL includes a signature + expiration)
- **signed cookies** (auth info in a cookie)

This lets the CDN enforce access without calling the origin for every request.

---

## 8) Dynamic acceleration (when it’s not just caching)

Some CDNs help even for dynamic requests:

- connection reuse (keep warm connections to origin)
- TLS termination at edge
- smart routing over private backbone networks
- request collapsing (coalescing identical origin requests)

This reduces latency and origin load even if the response isn’t cached long-term.

---

## 9) Common pitfalls (and how to avoid them)

### Pitfall A: Wrong cache key → privacy leak

Fix:

- don’t cache personalized pages at edge unless you’re very careful
- include correct headers/cookies in cache key

### Pitfall B: Stale content after deploy

Fix:

- versioned URLs for assets
- small TTLs for HTML

### Pitfall C: Cache stampede at the edge

If a very popular object expires and many users request it at once, the CDN can flood your origin.

Fix:

- “stale-while-revalidate” behavior (serve stale while refreshing)
- request collapsing at edge
- pre-warm hot objects

### Pitfall D: Cost surprises (egress)

CDNs can reduce origin egress, but CDNs themselves cost money too.

Fix:

- cache the biggest objects
- compress content
- tune TTLs
- monitor cache hit ratio and egress costs

---

## 10) What to monitor

- cache hit ratio (overall and per path)
- origin request rate (should drop with caching)
- edge latency vs origin latency
- error rates (4xx/5xx)
- invalidation/purge success and delays

---

## 11) Interview-ready summary

“We use a CDN to cache and serve content from edge POPs close to users, reducing latency and origin load. We control caching with cache keys and cache-control headers, and use versioned URLs for assets to avoid mass invalidations. For large files/video we rely on range requests and segment delivery. For private content we use signed URLs/cookies. We monitor cache hit ratio, origin load, and egress costs.”

