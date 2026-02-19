# CAP theorem (practical) — simple explanation

CAP is about tradeoffs during network partitions.

Practical statement:

> When there is a **network partition** (some servers can’t talk to others), a distributed system must choose between **Consistency** and **Availability**.

You don’t “choose two of three all the time.” CAP is about what you do **during partitions**.

---

## 0) The default way to apply CAP in real systems

Most real products are mixed:

- choose **CP behavior** for correctness-critical actions (payments, inventory, permissions)
- choose **AP behavior** for experience/freshness actions (feeds, analytics, telemetry)

The senior move is to tie this to user impact:

- “It’s okay to show a slightly stale feed.”
- “It’s not okay to double-charge.”

---

## 1) The three letters in plain English

### C — Consistency

“Everyone sees the same latest value.”

If you write `x = 5`, a later read returns `5` (or the system errors).

### A — Availability

“Every request gets a response.”

The system returns something (success or a non-error response) even if some nodes can’t talk.

### P — Partition tolerance

“The network can break.”

Messages can be lost/delayed, regions can be disconnected, or racks can fail.

In real distributed systems, partitions happen. So **you must tolerate partitions**. That means the real choice is usually between **C and A** when P happens.

---

## 2) Example: two replicas with a network partition

You have two database replicas: Left and Right.

Now the network link breaks:

- Clients can still reach Left.
- Clients can still reach Right.
- Left and Right cannot sync.

Now a client writes `balance = 100` to Left, and another client reads `balance` from Right.

You must choose:

### Option 1: Choose Consistency (CP behavior)

Right replica will refuse the read (or return an error) because it can’t confirm it has the latest data.

- **Pro**: you never serve wrong/stale data for that read.
- **Con**: some users see errors or downtime during the partition.

### Option 2: Choose Availability (AP behavior)

Right replica will return whatever it has (maybe old data).

- **Pro**: users get responses even during partitions.
- **Con**: some users see stale or conflicting data; you must resolve it later.

This is CAP in practice.

---

## 3) CP vs AP: what it means for product behavior

### CP (Consistency + Partition tolerance)

Systems that prefer consistency will:

- block/deny some requests during partitions
- reduce availability to protect correctness

Good for:

- payments and ledgers
- inventory reservation (avoid oversell)
- auth and permissions (security-sensitive)

### AP (Availability + Partition tolerance)

Systems that prefer availability will:

- keep serving responses during partitions
- accept staleness/conflicts and reconcile later

Good for:

- news feeds and timelines
- likes/views counters (often approximate)
- analytics dashboards
- logging/telemetry pipelines

---

## 4) “Most systems are mixed”

Real products aren’t purely CP or AP. They are usually **CP for critical invariants** and **AP for everything else**.

Example: E-commerce

- “Charge customer” → CP (must be correct)
- “Update ‘people also bought’ recommendations” → AP (can be stale)

This is a great interview answer because it ties CAP to **UX and risk**.

---

## 5) How systems get “mostly consistent” without becoming unavailable

Even if you prefer consistency, you can improve availability by designing the product behavior carefully.

Techniques:

- **Graceful degradation**:
  - show cached/stale data with a warning
  - disable non-critical features temporarily (“brownout”)
- **Read-your-writes**:
  - route a user’s reads to the region/leader that accepted their write
- **Local reads with bounded staleness**:
  - allow reads if replica lag is within a threshold (e.g., ≤ 1s)
- **Retry later**:
  - return a “please retry” error for consistency-critical operations

---

## 6) CAP vs “consistency models” (don’t mix them up)

CAP is about what you do during **network partitions**.

Consistency models are about what reads can see *in general*:

- linearizable (strong)
- sequential
- causal
- eventual

CAP is one lens. Consistency models are a deeper toolbox.

---

## 7) Interview-ready talk track

“CAP says that during network partitions, you can’t have both strong consistency and availability. For money/inventory we choose CP behavior: we’d rather fail requests than serve wrong answers. For feeds/analytics we choose AP behavior: we keep serving, accept staleness, and reconcile later. Most systems are a mix depending on the user impact.”



