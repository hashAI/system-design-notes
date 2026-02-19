# Design a URL shortener — full, beginner-friendly system design

Goal: build a service like `bit.ly`:

- user gives a long URL
- system returns a short link like `https://sho.rt/Ab3K9`
- when someone visits the short link, they get redirected to the original URL

This is a classic interview problem because it’s simple on the surface but has real scaling and reliability tradeoffs.

---

## 1) Requirements (make it concrete)

### Functional requirements (must-have)

- Create a short URL from a long URL
- Redirect: `GET /{code}` returns a 301/302 redirect to the long URL

### Nice-to-have (optional if time)

- Custom alias (user chooses the code)
- Expiration time (link stops working after a date)
- Basic analytics (click count, last clicked time)
- Abuse prevention (spam/malware links)

### Non-functional requirements

- **Very low redirect latency** (redirects are the hot path)
- **High availability** (people expect links to work)
- **Massive read scale** (redirects >> creates)
- **Durability** (mappings shouldn’t disappear)

### Out of scope (say this in interviews)

- Full user account system
- Advanced analytics and dashboards
- Deep anti-malware scanning pipeline (mention it as future work)

---

## 2) Back-of-the-envelope scale (so decisions make sense)

Assume:

- 100M redirects/day
- 1M new short links/day
- peak factor 10×

Approx:

- Redirect average QPS ≈ 100M / 86,400 ≈ 1.2k QPS, peak ≈ 12k QPS
- Create average QPS ≈ 1M / 86,400 ≈ 12 QPS, peak ≈ 120 QPS

Key takeaway:

- **redirect is read-heavy and latency-critical**
- **create is low QPS**, but must be correct (unique code)

---

## 3) API design

### Create short URL

`POST /v1/shorten`

Request:

- `long_url` (string)
- `custom_alias?` (string)
- `expire_at?` (timestamp)

Response:

- `short_url`
- `code`

Important:

- Add an **idempotency key** (optional but strong signal) so retries don’t create duplicates.

### Redirect

`GET /{code}`

Response:

- `301` or `302` with `Location: <long_url>`

301 vs 302:

- **301** (permanent): browsers/caches may store it (good if mapping never changes)
- **302** (temporary): safer if mappings can change or expire

Most systems use:

- 302 by default, or
- 301 for stable links

---

## 4) Data model

Minimum table:

- `url_mapping`
  - `code` (primary key)
  - `long_url`
  - `created_at`
  - `expire_at` (nullable)
  - `owner_id` (optional)

Optional analytics:

- `url_stats`
  - `code` (primary key)
  - `click_count`
  - `last_clicked_at`

Or raw events:

- `click_events(code, ts, ip_hash, user_agent_hash, ...)` (usually async)

---

## 5) High-level architecture (simple and scalable)

### Redirect path (hot path)

1. Client requests `GET /{code}`
2. CDN/edge (optional) checks cache
3. Load balancer sends to redirect service
4. Redirect service:
   - checks Redis cache for `code -> long_url`
   - on miss: reads DB, populates cache
5. Return 301/302 redirect

Why this works:

- Redirect service is **stateless**, so you can scale it horizontally.
- Redis absorbs most reads, protecting the DB.

### Create path (write path)

1. Client calls `POST /v1/shorten`
2. Shorten service:
   - validates URL
   - generates a unique code (or checks custom alias)
   - writes mapping to DB
   - writes to cache (optional warm)
3. Returns the short URL

---

## 6) Code generation (the core design decision)

You need a short “code” that:

- is unique
- is short enough to be convenient
- can be generated fast at scale

### Option A: Random code (common)

Generate a random Base62 string (a–z, A–Z, 0–9).

- Pros: hard to guess, no coordination for sequential IDs
- Cons: you must handle collisions (rare but possible)

Collision handling:

- generate code
- try insert
- if conflict, generate a new one

Length choice:

- Base62 has 62 characters.
- 7 chars → $62^7 \approx 3.5 \times 10^{12}$ possibilities (huge).

### Option B: Auto-increment ID → Base62

- DB generates an increasing integer ID.
- Convert to Base62 to get a short code.

Pros:

- no collisions
- very simple mapping

Cons:

- predictable codes (can be enumerated)
- needs a centralized ID generator (or careful distributed ID design)

### Option C: Snowflake-like IDs → Base62

Generate a distributed unique ID (timestamp + machine id + sequence), then encode.

Pros:

- distributed, scalable
- mostly ordered

Cons:

- longer codes than pure random for same collision risk

Interview-friendly answer:

- “Use random Base62 codes with collision checks, or Snowflake IDs if we want deterministic uniqueness.”

---

## 7) Caching strategy (what, where, and TTL)

Cache key:

- `code -> long_url`

TTL choice:

- If links never change: long TTL (days/weeks) is fine.
- If links can expire: set TTL to match `expire_at`.

Stampede protection:

- TTL jitter (so many keys don’t expire at once)
- single-flight refresh per code

Hot links:

- very popular codes should be served from:
  - edge cache (CDN) if safe
  - Redis

---

## 8) Handling expiration

Approach:

- Store `expire_at` in DB.
- For redirect:
  - if expired → return 404 (or a “link expired” page)
- For cache:
  - set TTL to expire near `expire_at`

Optional cleanup:

- background job to delete expired mappings (not required for correctness)

---

## 9) Analytics (do it async)

Redirect must be fast. Don’t write to the database synchronously on every redirect.

Instead:

1. Redirect service emits an event: `click(code, ts, …)` to a queue/stream (Kafka).
2. Analytics consumer aggregates:
   - increment click counters
   - compute daily stats
3. Store results in a separate table/store.

Why this is better:

- redirects stay low latency
- analytics pipeline can be scaled independently
- you can sample events if volume is huge

---

## 10) Abuse and security (keep it simple)

Real URL shorteners are abused for phishing and spam. In interviews:

- mention **rate limiting** on create endpoint
- block obvious bad patterns (very long URLs, known malicious domains)
- optional: async “safe browsing” checks

Also:

- validate URL scheme (`http/https`)
- avoid open redirect vulnerabilities beyond the intended behavior

---

## 11) Failure modes (how the system behaves when things break)

### Cache down

- Redirect service falls back to DB.
- Protect DB:
  - use rate limiting/load shedding if needed
  - serve stale edge cache where possible

### DB down

- Redirects:
  - popular links may still work from cache
  - uncached links fail (503/404 depending on policy)
- Creates:
  - fail (return 503)

### Queue down (analytics)

- Redirects still work.
- Analytics becomes delayed (acceptable).

This is a great example of designing **non-critical work** to be async and failure-tolerant.

---

## 12) Scaling path (how it evolves as usage grows)

Start simple:

- one primary DB with good indexes
- Redis cache-aside
- stateless redirect service behind an LB

Then evolve:

- add read replicas if DB reads grow
- add sharding if the mapping table becomes too large / write heavy
- add CDN edge caching for the hottest codes

---

## 13) Interview-ready summary (30 seconds)

“Redirect is the hot path: we keep it fast with a stateless redirect service, Redis cache-aside, and optional CDN caching for hot codes. We store the mapping in a durable DB keyed by `code`. Codes can be random Base62 with collision checks (or Snowflake IDs encoded). Analytics is async via events so redirects stay low latency. We handle expiration via `expire_at` and cache TTLs, and protect the system with rate limiting on link creation.”

