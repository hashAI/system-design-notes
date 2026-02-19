# Observability — metrics, logs, traces (what to add to every design)

Observability answers three questions:

- **Is it working?** (metrics)
- **What happened?** (logs)
- **Why is it slow/broken?** (traces)

This page gives a practical checklist you can apply in interviews and production designs.

---

## 1) The golden signals (simple and powerful)

For most services, you should track:

- **Latency** (p50/p95/p99)
- **Traffic** (QPS/RPS)
- **Errors** (4xx/5xx, exception rates)
- **Saturation** (CPU, memory, thread pools, queue depth, DB connections)

If you mention these in interviews, you’ll sound senior immediately.

---

## 2) Metrics (what to measure)

### Service-level metrics

- request count by route + status
- latency histogram per route
- error rates per route
- retries/timeouts/circuit breaker opens

### Dependency metrics

- cache hit ratio and latency
- DB query latency and connection pool saturation
- queue lag and processing time
- third-party provider success rate

### Business/product metrics (often forgotten)

- checkout conversion rate
- payment failure rate
- notification delivery success
- feed freshness

Senior signal: tie technical health to user/business outcomes.

---

## 3) Logs (make them useful)

### Structured logs

Prefer structured logs (JSON) so you can filter reliably.

Include fields like:

- timestamp
- service
- env/region
- request_id / trace_id
- user_id/tenant_id (careful with privacy)
- route
- status code
- error code/category

### Log levels

- debug: high volume, short retention
- info: key events, moderate volume
- warn/error: always keep longer

### PII hygiene

Don’t log:

- passwords
- full credit card numbers
- full tokens

Mask sensitive values.

---

## 4) Tracing (distributed requests)

Tracing is critical in microservices.

### What to propagate

- `trace_id`
- `span_id`
- correlation/request id

### What to trace

- gateway entry span
- each service hop span
- DB/cache calls spans (or at least timing)

### Sampling

You can’t store 100% traces at scale.

Use:

- head sampling (simple)
- tail sampling (better for errors/slow requests)

---

## 5) Dashboards (what to include)

At minimum, per service:

- golden signals dashboard
- dependency health dashboard

Per critical workflow (checkout, signup, payments):

- end-to-end funnel metrics
- error breakdown
- latency breakdown by stage

---

## 6) Alerting (avoid noisy alerts)

### Prefer symptom-based alerts

Examples:

- “p99 latency > threshold for 10 minutes”
- “error rate > 1%”
- “SLO burn rate is too high”

### Avoid alerting on every metric spike

If you alert on CPU alone, you’ll get false positives.

Better:

- alert on user impact:
  - errors + latency + saturation

### Routing

- page for user-impacting issues
- ticket for non-urgent capacity warnings

---

## 7) Debugging playbook (how you’d use observability)

If an alert fires:

1. Check dashboards:
   - latency, errors, saturation
2. Identify the hot route/tenant/region.
3. Use traces:
   - where time is spent (which dependency/hop)
4. Use logs:
   - find error patterns and correlation IDs
5. Mitigate:
   - shed load, disable features, rollback, failover

---

## 8) Examples (what to say per system type)

### Caching systems

- hit ratio
- evictions
- hot keys
- p99 latency

### Queues

- backlog depth
- consumer lag
- retry rate
- DLQ size

### Databases

- query latency distributions
- slow queries
- lock contention
- replication lag

---

## 9) Interview-ready summary

“For observability, we instrument golden signals per service (latency, traffic, errors, saturation), and we collect structured logs with correlation IDs and safe PII handling. We use distributed tracing across hops and sample intelligently. We build dashboards per service and per critical user journey, and we alert on symptom-based SLO burn rates rather than noisy resource metrics. This gives us fast debugging loops: alerts → dashboards → traces → logs → mitigation.”

