# Design Search Autocomplete — full system design

Autocomplete feels simple, but it’s a great system design question because it requires:

- very low latency (tens of milliseconds)
- high QPS (every keystroke can trigger a request)
- ranking by popularity and personalization
- frequent updates (trends change)

This design focuses on:

> Given a prefix like “app”, return suggestions like “apple”, “app store”, “application”.

---

## 1) Requirements

### Functional requirements

- Prefix suggestions for a query string
- Ranked suggestions
- Support multiple languages/locales (optional)
- Update suggestions based on:
  - query logs (popularity)
  - content corpus (e.g., product names) (optional)
- Basic typo tolerance (optional)
- Personalization (optional)

### Non-functional requirements

- Very low latency (p95 in ~20–50ms is common target)
- High availability
- High QPS (keystrokes)
- Cost-aware (cannot hit heavy systems per keystroke)

---

## 2) Scale estimation (example)

Assume:

- 10M DAU
- 20 searches/day/user
- average 5 keystrokes per search that trigger autocomplete

Autocomplete requests/day:

- 10M × 20 × 5 = 1B requests/day

Average QPS:

- 1B / 86,400 ≈ 11.6k QPS

Peak factor 10×:

- ~116k QPS peak

Key takeaway:

- must be served from memory/edge-friendly systems
- cannot do expensive DB queries per keystroke

---

## 3) High-level architecture

### Online serving path

Client → API Gateway → Autocomplete service → in-memory index/trie → return top N suggestions

Caching:

- cache hot prefixes (like “a”, “ap”, “app”)
- CDN edge caching is possible for anonymous/global suggestions with careful cache keys

### Offline pipeline (building the suggestion model)

Query logs + click logs + content corpus → aggregation/ranking → build data structure (trie/index) → publish to serving fleet

---

## 4) API design

`GET /v1/autocomplete?q=app&locale=en_US&limit=10`

Response:

- `suggestions: [{ text, score, type }]`

Optional:

- `user_id` (if personalized) (usually derived from auth token)

---

## 5) Data sources (where suggestions come from)

Common sources:

- **query logs**: what users type
- **click logs**: which suggestions led to a click (strong ranking signal)
- **catalog/content**: product names, usernames, tags (if applicable)

For search engines, query logs are usually the biggest driver of autocomplete quality.

---

## 6) Core serving data structure options

### Option A: Trie (prefix tree) in memory

Store characters as a tree. Each node represents a prefix.

For each node, store:

- top K suggestions for that prefix (precomputed)

Serving is fast:

- walk the trie by characters in the prefix → return top K list

Pros:

- extremely fast (in-memory)
- predictable latency

Cons:

- memory-heavy if you store too many nodes and suggestions

### Option B: Search engine (Elasticsearch/OpenSearch) with n-grams

Index suggestion terms with edge n-grams.

Pros:

- flexible matching, typo tolerance
- easier updates

Cons:

- higher latency and cost than an in-memory trie for very high QPS

### Practical hybrid (common)

- Use in-memory trie for top prefixes/queries (hot set)
- Use search engine fallback for long-tail or typo-tolerant queries

---

## 7) Building the suggestions (offline pipeline)

### Step 1: Collect events

- user typed query
- user clicked a result

### Step 2: Aggregate popularity

Compute counts by:

- query string
- locale
- device type (optional)
- time window (recent vs historical)

Example:

- `count_7d`, `count_1d`, `count_1h`

### Step 3: Rank

A simple, effective scoring function:

- score = \(a \cdot \text{count_7d} + b \cdot \text{count_1d} + c \cdot \text{count_1h}\)
- plus click-through rate boosts

### Step 4: Generate prefix maps

For each suggestion term, generate prefixes:

- “apple” → “a”, “ap”, “app”, “appl”, “apple”

Store top K per prefix.

### Step 5: Publish to serving

Publish as:

- a versioned snapshot file (e.g., in object storage)
- serving nodes download and swap atomically

Atomic swap:

- build new trie in memory
- switch pointer to new structure
- old one is freed after

---

## 8) Updates and freshness

You can rebuild:

- hourly for trending
- daily for baseline

Two-tier model:

- long-term popularity (stable)
- short-term trending boosts (fresh)

Serving merges:

- base suggestions + trending overlay

---

## 9) Personalization (optional)

Personalization should not blow up latency.

Approaches:

- small per-user embedding/interest categories
- re-rank top 20 results using user signals

Keep candidate generation global; personalize the ranking lightly.

---

## 10) Typos (optional)

Autocomplete is prefix-based, so typos are tricky.

Options:

- use search engine fallback that supports fuzzy matching
- support “did you mean” after full query, not per keystroke

Interview-friendly:

- “We focus on prefix correctness; typo tolerance can be handled by the main search engine rather than autocomplete.”

---

## 11) Caching strategy

- Cache hot prefixes in memory on the service (LRU)
- Cache first character prefixes heavily (they’re extremely hot)
- Add request debouncing on client:
  - don’t send requests for every keystroke; wait ~50–150ms

Client debouncing is a big real-world performance win.

---

## 12) Failure modes

- Offline pipeline delayed:
  - serving keeps last snapshot
  - quality may be slightly stale but system stays up
- One region down:
  - route to another region or degrade to non-personalized
- Search engine fallback down:
  - serve trie-only suggestions

---

## 13) Observability

Metrics:

- p95/p99 latency
- QPS
- cache hit ratio
- suggestion click-through rate (quality)
- pipeline lag (freshness)

---

## 14) Interview-ready summary (30 seconds)

“Autocomplete is extremely latency and QPS sensitive, so we serve suggestions from an in-memory trie where each prefix node stores precomputed top K suggestions. An offline pipeline aggregates query/click logs, computes popularity and trending scores, builds versioned snapshots, and serving nodes swap them atomically. We cache hot prefixes, apply client-side debouncing to reduce QPS, and optionally use a search engine fallback for long-tail and typo-tolerant matching. We monitor latency, cache hit ratio, and click-through rate.”

