# Consistency models — simple guide (strong vs eventual)

“Consistency” answers a simple question:

> If I write something, what can other people read — and when?

In distributed systems, different replicas may not agree instantly, so we choose a consistency model that matches the product needs.

This guide explains the common models in beginner-friendly language and how to talk about them in interviews.

---

## 0) The default consistency plan (what you’d propose in most designs)

- Use **strong consistency** for:
  - money/ledger
  - inventory reservations
  - auth/permissions
- Use **eventual consistency** for:
  - feeds/timelines
  - analytics and counters
  - derived views (search indexes, recommendations)
- Add **session guarantees** (read-your-writes, monotonic reads) to make UX feel correct even when backend is eventual.

---

## 1) Start with the user experience

Before naming any model, ask:

- If I post a message, should I see it immediately?
- If I update my profile photo, should others see it immediately?
- If I buy an item, can the system ever show it as still available?

Different features can use different consistency.

Simple rule:

- **Money, inventory, permissions** → stronger consistency
- **Feeds, analytics, counters** → eventual consistency is usually fine

---

## 2) Strong consistency (linearizable) — “one truth”

### What it means (plain English)

It behaves like there is a single up-to-date copy of the data.

- After a write completes, all later reads return the latest value.

### Why it’s nice

- easiest to reason about
- fewer weird user bugs

### Why it’s hard/expensive

- usually requires coordination between replicas
- adds latency (especially across regions)
- during partitions, you often lose availability (CP behavior)

### Good for

- payments/ledger entries
- inventory reservations
- uniqueness constraints (e.g., “username must be unique”)

---

## 3) Eventual consistency — “it will settle”

### What it means

Replicas can be temporarily different, but they eventually converge.

So a read might be stale for a short time.

### Why it’s popular

- high availability (keeps serving during many failures)
- can be low latency (read locally)
- scales well in geo-distributed systems

### The cost

- users may see stale or out-of-order information
- you must handle edge cases: duplicates, conflicts, and reordering

### Good for

- news feeds and timelines
- view counters/likes (often approximate)
- logging/telemetry pipelines

---

## 4) “Session” guarantees (very practical and easy to remember)

These are user-friendly consistency promises that make products feel correct even if the backend is eventually consistent.

### A) Read-your-writes

After you write, you should see your own write.

Example:

- You update your bio → you should see your new bio immediately.

Implementation ideas:

- route your reads to the leader/region where you wrote
- stick to a replica that has caught up (track replication position)
- serve from your write path cache

### B) Monotonic reads

Once you’ve seen a newer value, you should not later see an older value.

Example:

- You refresh a feed and see “Post A”.
- On the next refresh, it shouldn’t disappear due to reading from a stale replica.

### C) Writes-follow-reads

If you read an item and then update it, your update should be based on at least what you saw.

This helps avoid “lost updates.”

### D) Causal consistency (simple version)

Cause comes before effect.

Example:

- If you see “Alice replied to Bob,” you should also be able to see Bob’s original message.

This is very useful for chat and timelines.

---

## 5) Consistency and replication lag (the real-world cause of staleness)

With leader–follower replication:

- writes go to leader
- followers apply changes after some delay

That delay is **replication lag**.

This is why:

- reading from replicas can be stale
- multi-region reads can be stale

Simple mitigation:

- read from leader for critical reads
- read from replicas for non-critical reads

---

## 6) Consistency vs latency (a common tradeoff)

If you require strong consistency across regions, you often pay cross-region round-trip time on the write/read path.

If you accept eventual consistency, you can:

- write locally, replicate async
- read locally (faster)

Beginner-friendly way to explain it:

- Strong consistency is “everyone agrees before answering.”
- Eventual consistency is “answer now, agree shortly after.”

---

## 7) Common problems in eventually consistent systems (and how to handle them)

### A) Duplicate events

With retries/queues, the same event may be processed twice.

Fix:

- use idempotency keys
- dedupe by unique message ID

### B) Out-of-order updates

Update #2 might arrive before update #1.

Fix:

- include version numbers / timestamps
- use last-write-wins only if acceptable

### C) Conflicting writes

Two users update the same record in different regions.

Fix:

- choose a conflict resolution strategy (domain-specific)
- or route writes to a single leader for that key

---

## 8) Interview-ready summary (short and correct)

“Strong (linearizable) consistency behaves like one up-to-date copy, which is easiest to reason about but adds latency and can reduce availability during partitions. Eventual consistency allows temporary staleness but improves availability and geo-latency. In many products we mix: strong consistency for money/inventory/permissions, and eventual consistency for feeds/analytics. We often add session guarantees like read-your-writes and monotonic reads to make UX feel correct.”

