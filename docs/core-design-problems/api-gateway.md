# Design an API Gateway — full system design

An API gateway sits at the front of your backend. It’s the “front door”:

- it receives client requests
- applies cross-cutting policies (auth, rate limits, routing)
- forwards to internal services

In interviews, gateways are a great place to show senior thinking about:

- reliability (timeouts/retries/circuit breakers)
- security (authn/authz, WAF)
- operability (metrics/tracing)

---

## 1) Requirements

### Functional requirements

- Route requests to the right backend service:
  - by host/path/method/headers
- Authentication:
  - validate tokens (JWT or opaque tokens)
  - attach identity context (user_id, tenant_id)
- Authorization (basic):
  - enforce route-level access policies (optional; often belongs in services too)
- Rate limiting and quotas
- Request/response transformations (optional):
  - add/remove headers
  - protocol translation (REST ↔ gRPC) (optional)
- Canary releases and traffic splitting:
  - route 1% to new version, then ramp up

### Non-functional requirements

- Low latency overhead (gateway should be fast)
- High availability (don’t become a single point of failure)
- Safe overload behavior (don’t amplify failures)
- Observability (metrics/logs/traces)
- Config updates without downtime

---

## 2) Where the gateway sits

Typical flow:

Client → CDN/WAF → **API Gateway** → internal services → databases/queues

If you already have a CDN/WAF:

- push simple protections outward (DDoS, simple rate limits)
- keep gateway focused on routing/auth/policy

---

## 3) Architecture: data plane vs control plane

### Data plane (fast path)

Runs on gateway instances and does per-request work:

- routing
- token validation
- rate limiting
- timeouts/retries (carefully)
- request logging/metrics

Must be:

- stateless
- horizontally scalable

### Control plane (slow path)

Manages config and policies:

- routes and service discovery
- auth configuration (JWKS keys, token introspection endpoints)
- rate limit rules
- canary rules
- certificates

Distributes config to data plane via:

- push (watchers) or
- pull (periodic refresh)

Senior signal: mention “control plane changes often, data plane must be stable and fast.”

---

## 4) Routing and service discovery

### Routing rules

Examples:

- `GET /v1/orders/*` → `orders-service`
- `POST /v1/payments/*` → `payments-service`
- `Host: api.eu.example.com` → EU region services

### Service discovery

Gateway needs to know where services live:

- static config (small systems)
- service registry (Consul, etcd)
- Kubernetes service discovery

Health checks:

- only route to ready instances
- outlier detection to eject bad instances

---

## 5) Authentication and authorization

### Authentication (authn)

Two common token styles:

#### JWT (self-contained)

- gateway validates signature using JWKS keys
- fast (no network call)
- risk: token revocation is harder (needs short TTL or revocation lists)

#### Opaque tokens

- gateway calls auth service to introspect token
- supports revocation centrally
- adds latency and a dependency

Practical hybrid:

- JWT with short TTL + refresh tokens
- plus a “blocklist” cache for urgent revocations

### Authorization (authz)

Even if gateway enforces route access, services should still enforce authorization because:

- internal calls can bypass gateway
- defense in depth

Gateway can enforce coarse rules:

- “only admin tokens can access /admin/*”

Services enforce fine-grained rules:

- “user can only access their own order.”

---

## 6) Rate limiting and quotas

Gateway is the best place for rate limits because it’s early and consistent.

Typical limits:

- per IP for unauthenticated traffic
- per user for authenticated traffic
- per tenant/api key for B2B
- per route for expensive endpoints

Implementation:

- local token bucket for smoothing
- Redis-backed counters for distributed quotas

---

## 7) Timeouts, retries, and circuit breakers (avoid making outages worse)

### Timeouts

Always set:

- connect timeout
- request timeout
- per-try timeout (if retrying)

### Retries (bounded)

Retry only when:

- request is idempotent (GET, PUT with idempotency keys)
- error is transient (timeout, 502, 503)

Common safe policy:

- 0–1 retries max with jittered backoff

### Circuit breakers

If a backend is failing, stop sending traffic to it temporarily:

- prevents retry storms
- protects resources

### Hedged requests (advanced)

Send a second request after a small delay to reduce tail latency.

Use carefully:

- increases load
- only for safe idempotent reads

---

## 8) Traffic management (canary, blue/green, shadow)

### Canary

- route 1% of traffic to v2
- monitor errors/latency
- ramp up gradually

### Blue/green

- run two complete environments
- switch over quickly

### Shadow traffic

- duplicate requests to a new service without affecting responses
- useful for validating changes safely

---

## 9) Observability

Gateway is a great telemetry point.

Metrics:

- RPS by route, method, status code
- latency p50/p95/p99 by route and backend
- retries, timeouts, circuit breaker opens
- rate limit blocks

Tracing:

- generate/propagate correlation IDs
- add span for gateway processing

Logs:

- structured logs with route, user/tenant, backend

---

## 10) Security and abuse protection

Gateway commonly integrates:

- TLS termination (or mTLS internally)
- WAF rules
- request size limits
- header normalization
- bot protection signals

Be careful with:

- logging sensitive headers or bodies (PII)
- SSRF risks (if gateway allows arbitrary upstreams)

---

## 11) Failure modes

### Gateway overload

- autoscale
- shed load (429/503)
- prioritize critical endpoints

### Control plane outage

- data plane should keep running with last known good config
- config updates are paused

### Bad config rollout

- use validation and staged rollout
- allow quick rollback to last good config

---

## 12) Interview-ready summary (30 seconds)

“We deploy a stateless API gateway data plane that handles routing, token validation, rate limiting, and safe resilience policies (timeouts, bounded retries, circuit breakers). A separate control plane manages routes, policies, and certificates and rolls out config safely. The gateway integrates with service discovery and health checks, supports canaries/shadow traffic, and provides strong observability with per-route metrics and distributed tracing. Services still enforce fine-grained authorization for defense in depth.”

