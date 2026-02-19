# Design a News Feed / Timeline — full system design

Feeds are hard because they mix:

- huge read volume (everyone opens the app and scrolls)
- ranking/personalization
- freshness vs consistency tradeoffs
- fanout challenges (celebrities with millions of followers)

This design focuses on the core system design problem:

> “Show a user a ranked list of posts from accounts they follow, fast and reliably.”

---

## 1) Requirements

### Functional requirements

- Users can:
  - create posts
  - follow/unfollow users
  - fetch their home feed (timeline)
- Feed returns:
  - recent posts from followed accounts
  - ordered by time or ranking
  - paginated (infinite scroll)

### Nice-to-have

- likes/comments counts
- hide/mute/report
- “seen” tracking
- notifications for new posts
- personalization/ranking model

### Non-functional requirements

- Low latency feed reads (p95 < 200ms for first page is common target)
- High availability (stale is often acceptable, errors are not)
- Scalability to high read QPS
- Eventual consistency is acceptable for most feed contents

### Out of scope (unless asked)

- Full ML ranking pipeline details
- Ads system (can mention it fits into candidate generation + ranking)

---

## 2) Back-of-the-envelope scale (example)

Assume:

- 50M DAU
- each user opens feed 5 times/day
- each feed request returns ~30 items

Requests/day:

- 250M feed requests/day
- avg QPS ≈ 2,893
- peak 10× ≈ 29k QPS

Writes:

- posts/day might be much smaller (e.g., 20M/day)

Key takeaway:

- feed is **read-heavy**
- caching and precomputation matter

---

## 3) The two classic approaches

### Approach A: Fanout-on-read (pull model)

When user opens feed:

1. fetch list of accounts they follow
2. fetch recent posts from each followed account
3. merge/sort/rank

Pros:

- cheap writes (posting is easy)
- simple storage model (posts stored by author)

Cons:

- read is expensive (many queries, heavy merging)
- worst for users who follow many accounts

### Approach B: Fanout-on-write (push model)

When someone posts:

1. write post to posts store
2. push (fanout) post ID into each follower’s “inbox feed”

When user opens feed:

- read their inbox list of post IDs (fast)

Pros:

- very fast reads

Cons:

- expensive writes for high-fanout users
- big storage/write amplification

---

## 4) A practical hybrid (what real systems do)

Most large systems use a hybrid:

- **push model** for normal users
- **pull model** for celebrities (high-fanout accounts)

Why:

- pushes are great until one user has 50M followers

So you:

- push to followers for most authors
- for celebrities, don’t push; instead merge them at read time or maintain special caches

---

## 5) High-level architecture

### Core data stores

- **Follow graph store**:
  - “who does user U follow?”
  - “who follows user X?” (needed for fanout-on-write)
- **Post store**:
  - store post content and metadata
- **Feed inbox store** (for push model):
  - per-user list of post IDs (timeline candidates)
- **Cache**:
  - first page results and/or inbox list
- **Stream/queue**:
  - post events → fanout workers → inbox updates
- **Ranking service** (optional):
  - scores candidates and returns ordered list

### Read path (serving feed)

1. API request: `GET /v1/feed?cursor=...`
2. Fetch candidate post IDs:
   - from inbox store (push model)
   - plus “celebrity posts” merged (pull model)
3. Fetch post details (batch)
4. Rank/sort
5. Return page + next cursor

### Write path (posting)

1. user creates post
2. store post in post DB
3. publish “new post” event
4. fanout workers update follower inboxes (for non-celebs)

---

## 6) Data model (simple, scalable)

### Posts

- `posts(post_id PK, author_id, created_at, content, ...)`

Index:

- `(author_id, created_at)` for “author profile page”

### Follow graph

You often need two access patterns:

- list of who I follow: `following(user_id -> list of followee_id)`
- list of my followers: `followers(user_id -> list of follower_id)`

These can be stored in:

- sharded SQL, or
- key-value / wide-column stores (design by query)

### Feed inbox (push model)

- `feed_inbox(user_id -> ordered list of post_id with timestamps)`

Store choice:

- wide-column store (Cassandra) or key-value list-like structure

Partition key:

- `user_id`

Clustering key:

- time or sequence

---

## 7) Fanout workers (push model)

When a post is created:

1. fanout service queries follower list
2. partitions followers into batches
3. writes post_id into each follower’s inbox

Important:

- this is asynchronous
- backlog is acceptable (eventual consistency)

Handling spikes:

- queue provides buffering
- workers autoscale

---

## 8) Celebrity problem (high fanout)

If an author has tens of millions of followers, pushing into all inboxes is too expensive.

Common solutions:

### Option A: Pull celebrities at read time

When building a user’s feed:

- read inbox posts (from push model)
- also fetch latest posts from “celebrity authors you follow”
- merge and rank

### Option B: Precompute for segments

For very large systems:

- maintain popular/celebrity candidate lists per region/segment
- merge with user-specific inbox

---

## 9) Ranking and personalization (simple split)

Large feeds are often two stages:

### Stage 1: Candidate generation

Get a few hundred/thousand candidate post IDs from:

- inbox store
- celebrity merges
- “recommended” pool

### Stage 2: Ranking

Score candidates based on:

- recency
- relationship strength
- engagement signals
- user interests

Then return top N.

Interview-friendly approach:

- “We keep ranking separate so we can iterate ML models without rewriting storage.”

---

## 10) Pagination (cursor-based)

Use cursor pagination to avoid offset issues:

- cursor = `(last_seen_time, last_seen_post_id)`

Return:

- items
- `next_cursor`

This is stable under concurrent writes.

---

## 11) Caching strategy

Cache the first page aggressively:

- many users open feed repeatedly
- the first page is most viewed

Options:

- cache final feed response for 30–120 seconds
- cache inbox list for quick candidate fetch

Stampede protection:

- TTL jitter
- single-flight refresh for hot users

---

## 12) Consistency model (what users should expect)

Feeds are usually eventually consistent:

- you might not see a post immediately
- the order can change slightly

But you can add UX guarantees:

- “your own post appears immediately” (write-through to your own inbox)
- “don’t show duplicates”
- “don’t go backwards” (monotonic reads per session)

---

## 13) Failure modes

### Fanout pipeline is slow

- feed becomes slightly stale
- reads still work (inbox store has older items)

### Inbox store partial outage

- serve cached feed response
- fallback to pull model for some users (slower) if possible

### Ranking service down

- fallback to simple recency sort

Senior signal: define graceful degradation options.

---

## 14) Observability

Metrics:

- feed latency p95/p99
- cache hit ratio
- fanout queue lag
- inbox write throughput
- candidate size and ranking latency

Quality metrics (product):

- freshness (time from post to first appearance)
- duplication rate
- missing rate during incidents

---

## 15) Interview-ready summary (30 seconds)

“Feeds are read-heavy, so we optimize reads using a push model: on post we fanout post IDs into followers’ per-user inboxes stored in a feed store, and on read we fetch candidates and rank them quickly. To handle celebrities we use a hybrid: we don’t fanout high-fanout authors; instead we merge their recent posts at read time. We use cursor pagination, cache the first page, accept eventual consistency, and degrade gracefully by falling back to recency sort if ranking is down.”

