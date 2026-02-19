# Design a Logging & Monitoring System — full system design

“Logging and monitoring” is really three systems:

- **Logs**: high-cardinality text/events for debugging (“what happened?”)
- **Metrics**: numbers over time for alerting (“is it healthy?”)
- **Traces**: request-level timelines across services (“where is it slow?”)

This design shows how to ingest, store, query, and alert at scale with cost control.

---

## 1) Requirements

### Functional requirements

- Ingest:
  - application logs (structured)
  - system logs
  - metrics (counters, gauges, histograms)
  - distributed traces (spans)
- Query/search:
  - log search by time range + filters
  - dashboards for metrics
  - trace exploration and correlation to logs
- Alerting:
  - alert rules based on metrics
  - notification routing (PagerDuty/Slack/email)
- Retention:
  - configurable retention per data type (hot vs cold)
- Multi-tenancy:
  - separate tenants/services with access controls

### Non-functional requirements

- High ingestion throughput (spiky)
- Cost efficiency (full indexing is expensive)
- Low-latency queries for recent data (hot window)
- Reliability (don’t lose critical telemetry during incidents)
- Privacy/security (PII scrubbing, access control)

---

## 2) Back-of-the-envelope scale

Assume:

- 10k hosts
- each host emits 200 log lines/sec at peak
- average log line 300 bytes

Ingest:

- 2M lines/sec
- bandwidth ≈ 600 MB/s

Storage/day:

- 600 MB/s × 86,400 ≈ 51.8 TB/day (raw, before compression/replication)

Key takeaway:

- You cannot index everything forever.
- You need tiering, sampling, and careful schema choices.

---

## 3) High-level architecture

### Data flow (common pattern)

Agents/SDKs → Ingestion endpoints → Buffer/queue → Processing → Storage + Index → Query/alerting UI

### Components

- **Agents/collectors**:
  - Fluent Bit / Vector / OpenTelemetry collectors
- **Ingestion layer**:
  - HTTP/gRPC endpoints
  - authentication, basic validation, backpressure
- **Buffer**:
  - Kafka (common) for logs/traces
  - metrics can use remote-write to a TSDB ingest tier
- **Processing**:
  - parsing/enrichment
  - PII scrubbing
  - sampling (traces)
  - aggregation/rollups (metrics)
- **Storage**:
  - logs: object store for raw + search index for hot
  - metrics: TSDB (Prometheus/Cortex/Mimir-like)
  - traces: trace store (Tempo/Jaeger-like) + index
- **Query services**:
  - logs search
  - metrics query engine
  - trace query engine
- **Alerting**:
  - rule evaluation + routing

---

## 4) Logs pipeline (deep dive)

### Log format: structured logs

Encourage JSON logs with consistent fields:

- timestamp
- service
- env
- severity
- request_id/trace_id
- message
- attributes

This makes search and dashboards reliable.

### Ingestion

Agents batch and compress logs, then send to ingestion.

Backpressure:

- ingestion returns 429/503 when overloaded
- agents buffer locally (disk) with limits

### Storage strategy (hot vs cold)

Hot window (e.g., 7–14 days):

- store in an indexed search system (OpenSearch/Elasticsearch)
- fast queries and filtering

Cold window (e.g., 30–365 days):

- store compressed logs in object storage (S3/GCS)
- query via:
  - on-demand scan jobs
  - or a cheaper index (partitioned by time/service)

Key cost control:

- index only what you need for the hot window
- archive everything else cheaply

---

## 5) Metrics pipeline (deep dive)

Metrics are designed for alerting and dashboards.

### Types of metrics

- counters (monotonic): requests_total
- gauges: cpu_usage
- histograms: latency buckets

### Storage

Use a TSDB:

- efficient for time-range queries
- supports downsampling/rollups

Retention strategy:

- high-resolution (10s) for 7–14 days
- downsampled (1m/5m) for months

Alerting:

- run rule evaluation continuously
- prefer SLO burn-rate alerts over raw thresholds for mature systems

---

## 6) Traces pipeline (deep dive)

Traces connect the dots across microservices.

### Trace basics

- a **trace** is a request
- a **span** is a step within that request

### Sampling (critical for cost)

You rarely store 100% of traces.

Common strategies:

- head-based sampling (sample at ingress)
- tail-based sampling (sample after seeing duration/errors)

Tail-based is better for catching slow/error traces, but more complex.

Store:

- recent traces in a fast store
- older traces sampled heavily

---

## 7) Correlation: logs ↔ traces ↔ metrics

This is where the system becomes powerful.

Make sure:

- logs include `trace_id` and `span_id`
- metrics include labels like service/endpoint

Then you can:

- see an alert (metrics) → jump to traces → open logs for that trace

---

## 8) Multi-tenancy and access control

You must prevent:

- tenant A reading tenant B logs

Common model:

- partition by tenant_id
- enforce authz in query services
- encrypt sensitive data

---

## 9) Reliability and failure modes

### Ingestion overload

- buffer in Kafka
- shed load for non-critical logs
- prioritize error logs and metrics

### Kafka outage

- agents buffer locally (disk) up to a limit
- ingestion can fail-open/closed depending on criticality

### Search index overload

- drop debug logs first
- reduce indexing fields
- keep raw logs in object storage so you can recover later

---

## 10) What to monitor (meta-monitoring)

Monitor the monitoring system:

- ingestion rate and errors
- queue lag
- indexing throughput
- query latency and failures
- storage growth

If the observability system dies during an incident, you lose your best debugging tool.

---

## 11) Interview-ready summary (30 seconds)

“We ingest telemetry via agents/OTel collectors into a buffered pipeline (Kafka) with backpressure. Logs are stored as raw compressed data in object storage and indexed for a short hot retention window in a search cluster; metrics go to a TSDB with rollups; traces are sampled and stored with correlation IDs. Query services enforce multi-tenant access control and enable ‘metrics → traces → logs’ workflows. We control cost via tiered retention, sampling, and limiting indexed fields, and we monitor the pipeline itself (lag, ingestion errors, indexing throughput).”

