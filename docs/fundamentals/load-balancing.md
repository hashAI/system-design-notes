# Load balancing — deep dive

Load balancing distributes requests across multiple backends. The goals are:

- **Availability**: remove unhealthy backends quickly; survive instance/AZ failures.
- **Latency**: keep p95/p99 low by avoiding slow/saturated backends.
- **Throughput**: use all instances efficiently.
- **Operational safety**: support deploys, draining, and progressive rollouts.

---

## 0) The default load balancing plan (good in most designs)

- Use an **L7 load balancer** (or API gateway) in front of **stateless** service instances.
- Enable **readiness** checks for traffic gating; use **liveness** for restarts (usually by orchestrator).
- Routing algorithm: start with **P2C** or **least outstanding requests**; use weights if instances differ.
- **Timeouts** and **bounded retries** (0–1 retries) only for idempotent operations.
- Enable **connection draining** for rolling deploys.

---

## 1) What a load balancer actually does

At a minimum, a load balancer (LB) does two jobs:

1. **Select a backend** for an incoming connection/request.
2. **Maintain safety invariants**:
   - don’t send traffic to unhealthy backends,
   - don’t overload a subset of instances,
   - degrade gracefully during partial failures.

In modern systems, LBs often also do:

- TLS termination (or pass-through)
- request routing (path/host/header)
- authentication / WAF (sometimes separated)
- timeouts and retries (careful!)
- rate limiting and quotas
- request/response transformation
- observability (metrics/logs/tracing)

---

## 2) Where load balancing happens (layers)

### A) Client-side load balancing

**Client** (service client library) picks the backend.

- **Pros**
  - No extra network hop (lower latency).
  - LB logic can be specialized (e.g., gRPC pick-first/round-robin).
- **Cons**
  - Every client must implement membership/discovery logic.
  - Rolling out LB behavior changes requires updating many clients.
  - Harder to enforce centralized policies (global rate limiting, WAF).

Common in:

- gRPC microservices using service discovery + client libraries.
- Some cache clusters via consistent hashing in clients.

### B) DNS-based / GSLB (global load balancing)

DNS answers with different IPs based on:

- geography (geo-DNS),
- health checks,
- latency,
- weighted routing (canary / gradual shift).

**Pros**

- Simple and globally scalable.
- Can steer users to closest region.

**Cons**

- DNS caching (TTL) makes changes slow and non-deterministic.
- Clients may ignore TTLs.
- Not ideal for per-request routing decisions.

Often paired with:

- Anycast (BGP) for “closest” POP routing,
- multi-region active-active deployments.

### C) L4 load balancing (TCP/UDP)

Operates at transport layer. It doesn’t need to parse HTTP.

- **Use when**
  - you need maximum throughput/lowest overhead,
  - you’re balancing non-HTTP protocols,
  - you want TLS pass-through,
  - you want a simple, robust distribution of connections.

**Key property**: decisions are often per-connection, not per-request.

### D) L7 load balancing (HTTP/gRPC-aware)

Understands requests:

- route by host/path/headers,
- enforce auth policies,
- can retry idempotent requests,
- can do canary, traffic shadowing, header-based routing.

**Tradeoff**: higher CPU and complexity; can become a bottleneck if not scaled or tuned.

### E) Service mesh sidecars (east-west traffic)

In microservices, a sidecar proxy per service instance can perform:

- local load balancing,
- mTLS,
- retries/timeouts/circuit breakers,
- telemetry.

The “LB” becomes distributed.

Senior signal: mention **central policy** (control plane) vs **data plane** (sidecars) and the operational overhead.

---

## 3) Core algorithms (how a backend is chosen)

### Round robin (RR) and weighted RR

Cycle through instances. Weighted RR assigns more traffic to stronger nodes.

- **Pros**: simple, works well when requests are similar.
- **Cons**: ignores runtime load; can overload slower nodes.

### Least connections / least outstanding requests

Choose the backend with the fewest active connections or in-flight requests.

- **Pros**: adapts to uneven request costs.
- **Cons**: requires accurate tracking; can be fooled if long-lived connections dominate.

### Random with power of two choices (P2C)

Pick two random backends, choose the better one (lower load/latency).

- **Pros**: great balance/overhead tradeoff; widely used in large-scale systems.
- **Cons**: still needs some load signal.

### Latency-aware routing

Route to the instance with best recent latency.

- **Pros**: optimizes tail latency.
- **Cons**: can create positive feedback loops (hot instance looks good → gets more traffic → degrades).

Mitigation: smoothing, exploration, caps per instance.

### Consistent hashing

Route based on a key (user ID, session ID, cache key) so the same key maps to the same backend.

Used for:

- distributed caches (shard-by-key),
- sticky sessions (where you can’t avoid them),
- stateful sharding.

Key concepts:

- **hash ring** with **virtual nodes** to smooth distribution,
- **minimal remapping** on membership changes.

Tradeoffs:

- can create hotspots if key distribution is skewed,
- needs careful handling of “hot keys” (replicate/split).

---

## 4) Health checks (what “healthy” means)

### Liveness vs readiness

- **Liveness**: “process is running.” If failing, restart.
- **Readiness**: “safe to receive traffic.” If failing, remove from LB pool.

Readiness should reflect dependency state:

- warmed caches,
- DB connectivity,
- required configuration loaded.

### Active vs passive health checks

- **Active**: LB probes `/health` periodically.
- **Passive**: LB infers health from real traffic failures (timeouts, 5xx).

Best practice: use both, but avoid flapping:

- use consecutive-failure thresholds,
- use exponential backoff for re-adding instances,
- use outlier detection (eject a node with abnormal error rates).

### Load-aware routing (avoid saturated backends)

A backend can be alive but not safe to receive more traffic (queueing, GC pauses, downstream contention).

Signals to use:

- in-flight requests / queue length
- timeouts and error rate
- p95/p99 latency
- CPU saturation

Actions:

- reduce weight temporarily or eject outliers
- upstream admission control (concurrency caps)

---

## 5) Sticky sessions (when you can’t avoid state)

Sticky sessions bind a client to a backend, typically via cookie or consistent hash.

### Why stickiness is risky

- reduces load balancing effectiveness,
- increases blast radius when a node fails (many sessions reset),
- complicates scaling (rebalance causes churn),
- breaks in multi-region / failover scenarios.

### Preferred alternatives

- make service tier **stateless**,
- store session state in:
  - Redis (short-lived),
  - DB (durable),
  - signed client tokens (JWT) if appropriate.

If you must use stickiness, say:

- “We’ll keep state in a shared store and use stickiness only as an optimization.”

---

## 6) Timeouts, retries, and the retry storm trap

Gateways and L7 LBs often implement retries. Retries help transient errors but can amplify load during partial failures.

### The good

- retry **idempotent** requests on transient failures (connection reset, 502).
- use hedged requests (carefully) to reduce tail latency.

### Retry storms (how outages get worse)

If a backend is slow, retries amplify load, making it slower.

Mitigations:

- **bounded retries** (e.g., 1 retry max),
- **jittered backoff**,
- **circuit breakers** when error rate rises,
- **per-try timeouts** (don’t just extend total time),
- avoid retrying non-idempotent writes unless you have idempotency keys.

---

## 7) Connection management and keep-alives

Load balancing is often limited by connection behavior:

- HTTP keep-alive reduces handshake overhead.
- gRPC uses long-lived HTTP/2 connections.

Implications:

- L4 LB may distribute connections unevenly if clients create few long-lived connections.
- With gRPC, **connection-level RR** can lead to skew.

Mitigations:

- use multiple subchannels/connections per client,
- use request-level LB when appropriate,
- use P2C with per-connection load signals.

---

## 8) Global traffic management (multi-region)

When you operate across regions, you need to decide:

- **active-active** (serve in multiple regions concurrently),
- **active-passive** (hot standby),
- **hybrid** (read-local, write-primary).

Routing options:

- geo-DNS + health checks,
- Anycast to nearest POP,
- application-level routing via edge gateway.

Senior-level considerations:

- **failover time**: DNS TTL can be slow; Anycast can converge faster.
- **state**: where does user/session/data live?
- **consistency**: do you allow stale reads?

---

## 9) Overload behavior (load shedding and fairness)

When demand exceeds capacity, a good system fails *predictably*:

- **return 429** for rate limited clients,
- **return 503** when overloaded,
- enforce **priorities** (critical endpoints first),
- apply **admission control** (queue limits, concurrency caps),
- serve **stale** or **degraded** responses for non-critical reads.

Key idea: your LB/gateway is often the best place to enforce this because it’s at the edge of the system.

---

## 10) Observability: what to measure

At minimum:

- request rate (RPS/QPS)
- error rate (4xx vs 5xx)
- latency distribution (p50/p95/p99)
- backend selection distribution (is traffic skewed?)
- retries and timeouts
- health check flaps

High-signal debugging story:

- “If one instance has 10× the p99 latency, we eject it (outlier detection) and investigate GC pauses, noisy neighbors, or dependency contention.”

---

## 11) Common failure modes (and mitigations)

- **Uneven traffic distribution**
  - Causes: long-lived connections, small backend pools, bad hashing, weight misconfig.
  - Fix: P2C, more connections, better hashing, tune weights.

- **Health check mismatch**
  - Liveness passes but readiness should fail (dependency down).
  - Fix: separate endpoints, readiness gates.

- **Cascading failures**
  - LB retries too aggressively; backends saturate.
  - Fix: bounded retries, circuit breakers, load shedding.

- **Thundering herd on deployment**
  - All instances restart; LB flaps.
  - Fix: rolling deploys, connection draining, pre-warming, startup probes.

- **Hot key amplification**
  - One key drives load; LB keeps routing to same shard.
  - Fix: cache/replicate/split hot key, isolate hot partition.

---

## 12) Interview-ready summary (what to say in 20 seconds)

- Stateless service tier behind an L7 LB/gateway.
- Readiness checks + outlier detection; connection draining for deploys.
- Routing: P2C / least outstanding; consistent hashing when routing by key.
- Timeouts everywhere; 0–1 retries for idempotent operations; circuit breakers.
- Overload: admission control + 429/503; prioritize critical endpoints.

