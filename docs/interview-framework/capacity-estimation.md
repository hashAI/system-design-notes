# Back-of-the-envelope math (capacity estimation) — deep guide

Capacity estimation is not about being “right.” It’s about:

- picking **orders of magnitude** correctly,
- identifying **dominant costs/bottlenecks**,
- and using numbers to justify architecture decisions.

In interviews, good estimation changes your design: cache placement, partitioning, queueing, replication, and cost controls.

---

## 0) The mindset: estimate to decide

You’re rarely asked for precise capacity planning. You’re expected to answer questions like:

- “Will this DB melt at peak traffic?”
- “Do we need a cache?”
- “What’s the storage growth per day?”
- “Where do we shard first?”
- “Is multi-region feasible with this consistency requirement?”

If your numbers don’t change a decision, you’re either over-estimating detail or missing the point.

---

## 1) The 90-second estimation framework

When the interviewer says “assume some scale,” do this quickly:

1. **Pick a user base and activity rate**
   - MAU/DAU
   - actions per user per day
2. **Convert to requests/day**
3. **Convert to average QPS**
4. Apply a **peak factor** (usually 5–20×)
5. For each request type:
   - estimate **bytes in/out**
   - estimate **DB operations** (reads/writes)
6. Compute:
   - bandwidth at peak
   - storage growth per day
   - hot key/hot partition risks

Write the numbers on the board. Then explicitly say: “The bottleneck is likely X, so we’ll do Y.”

---

## 2) Core formulas and conversions

### QPS from requests/day

\[
\text{Average QPS} \approx \frac{\text{requests/day}}{86{,}400}
\]

Peak QPS:

\[
\text{Peak QPS} \approx \text{Average QPS} \times \text{peak factor}
\]

Peak factor is a proxy for diurnal patterns + burstiness. If unsure:

- consumer social apps: 10× is common
- enterprise workloads: 3–5× is common
- global products with follow-the-sun: lower peak factor

### Bytes to bandwidth

\[
\text{Bandwidth} \approx \text{bytes/request} \times \text{QPS}
\]

Convert:

- 1 KB = 1,000 bytes (interview math; don’t sweat 1024 vs 1000)
- 1 MB/s ≈ 8 Mbps

### Storage growth

\[
\text{storage/day} \approx \text{writes/day} \times \text{bytes/write}
\]

Then multiply by:

- replication factor (e.g., 3×)
- index overhead (often 20–100% depending on schema)
- metadata overhead (headers, pointers, etc.)

### Rule-of-thumb latencies (ballpark)

These vary wildly, but the *relative* ordering is what matters:

- In-process memory: **nanoseconds**
- Network hop in same AZ: **~0.1–1 ms** (plus tail)
- Redis/memcache: **~0.5–2 ms** (plus tail)
- SSD random read: **~0.1 ms** (device), but DB adds overhead
- DB query: **~1–20 ms** typical, can be 100ms+ if bad plan/contention
- Cross-region round trip: **tens to hundreds of ms**

The senior move: budget p99 latency across hops and leave room for retries.

---

## 3) A reusable “defaults table”

If the interviewer doesn’t care about precision, use consistent defaults:

- **DAU/MAU**: 10–30% depending on product
- **Peak factor**: 10×
- **Reads:writes**:
  - feeds/redirects: 100:1 to 1,000:1
  - messaging: closer to 10:1 (reads to writes)
  - logging: write-heavy
- **Typical payload sizes**:
  - API JSON response: 0.5–5 KB
  - chat message: 200–1,000 bytes
  - event log: 200–2,000 bytes

Use these to move quickly and focus on architecture.

---

## 4) How to estimate a system: worked examples

### Example A: URL shortener redirect path (read-heavy)

Assume:

- 100M redirects/day
- peak factor 10×
- response: mostly HTTP redirect; assume 1 KB effective bytes/request (headers etc.)

Compute:

- Average QPS ≈ 100,000,000 / 86,400 ≈ 1,157 QPS
- Peak QPS ≈ 11,570 QPS
- Bandwidth ≈ 1 KB × 11,570 QPS ≈ 11.6 MB/s ≈ 93 Mbps

What this implies:

- Bandwidth is non-trivial but manageable; **latency and tail latency** matter more.
- DB read IOPS can be the bottleneck if every redirect hits DB.
- You should propose:
  - edge/CDN caching for popular codes (if safe)
  - Redis cache-aside
  - origin DB optimized for key lookups (indexed PK)

### Example B: Notification system (write-heavy, async)

Assume:

- 10M notifications/day
- average payload: 2 KB
- peak factor 5×

Compute:

- Avg QPS ≈ 116 QPS, peak ≈ 580 QPS (not huge)
- Storage/day for request records ≈ 10M × 2 KB = 20 GB/day (source-of-truth + outcomes)
- If you keep 30 days: ~600 GB (before replication/index overhead)

What this implies:

- Not QPS-bound; **reliability and provider throughput** dominate.
- Use queues + retries + DLQ.
- Consider idempotency and dedupe keys.

### Example C: Logging ingestion (high throughput)

Assume:

- 50k hosts
- each host emits 200 log lines/sec at peak
- average log line 300 bytes

Compute:

- Lines/sec ≈ 50,000 × 200 = 10,000,000 lines/sec
- Ingest bandwidth ≈ 10M × 300 bytes/sec = 3 GB/s (massive)
- Storage/day ≈ 3 GB/s × 86,400 ≈ 259,200 GB/day ≈ 259 TB/day (before replication)

What this implies:

- You can’t fully index everything forever.
- You need:
  - aggressive sampling/filters
  - tiered retention (hot indexed, cold archived)
  - compression and batching
  - cost-aware design (object storage)

This is exactly why estimation matters: it changes architecture immediately.

---

## 5) Hotspots: the most common “gotcha”

Even if average QPS is fine, one key/partition can dominate.

### Hot keys

Examples:

- one viral URL short code
- one celebrity’s feed
- one shared resource ID

Mitigations:

- cache at edge + in-memory
- replicate hot items
- split counters (sharded counters)
- isolate hot partitions

### Hot partitions

If you partition by user ID, you’re fine until one user is huge.
If you partition by timestamp, you may create “current time” hotspots.

Mitigations:

- composite partition keys (e.g., `(user_id, bucket)` or `(time_bucket, hash)`)
- dynamic splitting (smaller cells, more shards)
- two-tier architectures (special-case celebrities)

---

## 6) Estimating database load (IOPS and query patterns)

Rather than guessing IOPS numbers, do *relative* reasoning:

- “If each request causes 2 cache lookups + 1 DB read on miss, and the miss rate is 5%, DB sees 0.05 × QPS reads.”

Example:

- Peak QPS = 20k
- cache hit rate = 95%
- DB reads ≈ 1k QPS

Then you can argue:

- 1k QPS reads might be okay for a single well-indexed primary, but p99 will still matter.
- add read replicas if needed (but discuss lag)

Senior signals:

- you separate **read path** and **write path** concerns
- you mention **indexing**, not just scaling hardware
- you talk about **tail latency** and contention

---

## 7) Latency budgeting (p99 thinking)

If the requirement is p99 = 200ms, you can’t spend 200ms on each hop.

Example budget:

- LB + gateway: 10ms
- service compute: 30ms
- cache: 10ms
- DB: 80ms
- network/tails: 40ms
- buffer for retries: 30ms

Then say:

- “Because retries consume budget, we must keep timeouts tight and only retry idempotent operations.”

---

## 8) Multi-region: bandwidth + consistency costs

Estimation can show when multi-region is hard:

- If every write must be strongly consistent across regions, you pay cross-region RTT on the critical path.
- If you accept eventual consistency, you can:
  - write locally, replicate async
  - read locally with bounded staleness semantics

Senior move: tie this to UX:

- “Payments must be correct; we accept some unavailability to keep strong invariants.”
- “Feed freshness can be slightly stale; we prefer availability.”

---

## 9) Interview-grade accuracy: what to include vs omit

Include:

- order-of-magnitude QPS
- peak factor
- storage/day and retention
- the dominant cost/bottleneck

Omit unless asked:

- precise CPU core counts
- exact cost dollars
- extremely detailed network packet overheads

If asked about compute sizing, keep it simple:

- “If one instance can serve ~2k RPS at p95 under our workload, we need 10 instances for 20k RPS plus 2× headroom.”

---

## 10) Quick checklist (use this under time pressure)

- What is the peak QPS for reads and writes?
- Which path is latency-critical?
- What’s the payload size → bandwidth at peak?
- What is storage growth/day and retention needs?
- What are the hotspots (hot keys/partitions)?
- Which component becomes the bottleneck first?
- What architecture choices directly address that bottleneck?

