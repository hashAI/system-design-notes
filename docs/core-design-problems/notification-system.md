# Design a Notification System (email/SMS/push/in-app) — full system design

Notification systems are “reliability systems.” Users tolerate a feed being slightly stale, but they notice missed OTPs, missing password reset emails, or delayed critical alerts.

This design covers:

- email, SMS, push, and in-app notifications
- user preferences and templates
- retries, provider failures, and idempotency
- observability and operational controls

---

## 1) Requirements

### Functional requirements

- Send notifications via:
  - email
  - SMS
  - push (APNs/FCM)
  - in-app (stored + retrievable)
- Support notification types:
  - transactional (OTP, password reset, receipt)
  - marketing (optional; stronger preference controls)
  - system alerts (SRE/on-call)
- Template rendering:
  - variables (name, order_id, etc.)
  - localization (optional)
- User preferences:
  - opt-out per channel/category
  - quiet hours / do-not-disturb (optional)
- Scheduling (send later) (optional)
- Dedupe/idempotency:
  - don’t send duplicates on retries

### Non-functional requirements

- High reliability and delivery guarantees (best effort, but strong engineering)
- Provider failover (Twilio down, SES throttling, APNs transient errors)
- Low latency for API acceptance (enqueue quickly)
- Scalable throughput
- Compliance (CAN-SPAM, GDPR), privacy, audit trails

### Out of scope (unless asked)

- Sophisticated marketing campaign engines
- ML-based send-time optimization

---

## 2) Back-of-the-envelope scale (example)

Assume:

- 50M users
- peak: 200k notifications/min during large event (~3.3k/sec)

Key implications:

- API must ingest quickly → async queue is required
- workers must autoscale
- providers have rate limits → throttling and backpressure are required

---

## 3) High-level architecture

### Core components

- **Notification API** (sync): validates request, checks entitlement, writes record, enqueues job
- **Template service**: stores templates, renders content
- **Preference service**: stores per-user preferences and compliance settings
- **Queue/stream**: Kafka / RabbitMQ / SQS
- **Worker fleet**: channel-specific senders (email worker, SMS worker, push worker)
- **Provider adapters**:
  - SES/SendGrid for email
  - Twilio for SMS
  - APNs/FCM for push
- **Notification store**:
  - notification records (status, attempts, provider response)
  - in-app notification storage
- **DLQ**: dead-letter queue for poison pills
- **Analytics/metrics** (optional): delivery rates, bounce rates

### Basic flow

1. Client calls Notification API.
2. API creates a `notification_request` row (idempotent).
3. API publishes a message to queue.
4. Workers consume, render template, apply preferences, send via provider.
5. Worker records outcome and retries if needed.

---

## 4) API design

### Send notification

`POST /v1/notifications:send`

Request:

- `idempotency_key` (required for reliability)
- `user_id` (or list for batch)
- `category` (transactional/marketing/system)
- `channels` (email/sms/push/in_app)
- `template_id`
- `variables` (key-value)
- `schedule_at?`

Response:

- `notification_id`
- `status = accepted`

Important:

- Response should not depend on provider success; it should mean “queued.”

---

## 5) Data model (minimum)

### Notification request table

- `notification_id` (PK)
- `idempotency_key` (unique with scope, e.g., per user or per client)
- `user_id`
- `category`
- `template_id`
- `variables` (JSON)
- `schedule_at`
- `created_at`
- `status` (accepted, processing, sent, failed, cancelled)

### Per-channel delivery attempts

- `notification_delivery(notification_id, channel, attempt, provider, provider_message_id, status, error_code, created_at)`

### In-app notifications

- `in_app_notifications(user_id, notification_id, created_at, read_at, payload...)`

Why separate deliveries:

- one notification can attempt multiple channels
- one channel can retry multiple times

---

## 6) Templates and rendering

Template content:

- subject/title
- body (html/text)
- localization keys (optional)

Rendering should be deterministic:

- use a safe template engine
- validate variables at creation time (fail early)

Example:

- Template: “Your OTP is {{code}}”
- Variables: `{ code: "123456" }`

---

## 7) Preferences and compliance

Two common layers:

### A) User preferences

- opt-out marketing emails
- allow transactional emails always (depending on product policy)
- channel selection (push only, email only)

### B) Compliance rules

Marketing requires stricter consent in many regions.

Interview-friendly statement:

- “We treat transactional and marketing separately; transactional is typically allowed for account safety, marketing requires opt-in and unsubscribe links.”

Quiet hours:

- if user has DND window, schedule for later (except critical security alerts)

---

## 8) Queuing strategy

### What goes on the queue

Message contains:

- notification_id
- channel(s) to process
- priority (transactional > marketing)
- scheduled time (or use delayed queues)

### Priority handling

Approaches:

- separate queues per priority
- one queue with priority field + consumer side scheduling

For critical notifications (OTP):

- dedicated queue + dedicated worker pool to protect latency

---

## 9) Retries, backoff, and DLQ (the reliability core)

### Retry policy

Most provider failures are transient:

- timeouts
- 5xx
- throttling (429)

Use:

- exponential backoff with jitter
- max attempts

Example:

- attempt 1: immediate
- attempt 2: 10s
- attempt 3: 60s
- attempt 4: 5m
- then DLQ

### Permanent failures (don’t retry)

- invalid email address
- blocked number
- user unsubscribed

Mark failed with reason.

### DLQ

If a notification keeps failing (poison message), move to DLQ:

- allows pipeline to keep flowing
- humans can inspect and replay after fix

---

## 10) Idempotency and dedupe (avoid double sends)

Duplicates happen because:

- client retries API call
- worker retries after timeout
- queue delivers at-least-once

Solutions:

### API-level idempotency

- `idempotency_key` unique constraint ensures only one `notification_id` is created.

### Provider-level dedupe (when possible)

- store `provider_message_id`
- if you retry, reuse the same provider idempotency token if provider supports it

### Worker-level idempotency

Before sending:

- check if that channel is already marked `sent` for this notification.

---

## 11) Provider failover

Providers can fail or throttle.

Design:

- adapter interface per channel
- configured primary + secondary providers

Failover logic:

- if provider returns throttle/5xx repeatedly → switch provider for a time window
- keep per-provider health metrics

Senior signal:

- “We track provider success rate and latency; we can route traffic dynamically during incidents.”

---

## 12) Scheduling / delayed delivery

Options:

- delayed queues (SQS delay, RabbitMQ delayed plugin)
- store `schedule_at` in DB and have a scheduler service enqueue when due

Keep it simple:

- if scheduling is required, implement a scheduler that scans due notifications and enqueues them.

---

## 13) Observability and operations

Metrics:

- accepted rate, sent rate, failed rate (by channel, provider, category)
- provider latency and error codes
- queue depth / lag
- retry counts
- DLQ growth
- bounce/complaint rate for email

Dashboards:

- OTP latency and success is a top SLO candidate.

Runbooks:

- provider outage (failover steps)
- spike handling (autoscale workers, throttle marketing)

---

## 14) Security and privacy

- protect PII (emails, phone numbers)
- encrypt in transit; encrypt sensitive at rest
- audit logs for notification sends (especially security-sensitive)
- prevent abuse:
  - rate limit OTP requests
  - detect repeated sends to same destination

---

## 15) Interview-ready summary (30 seconds)

“We ingest notification requests via an API that validates input, enforces idempotency, stores a durable request record, and enqueues jobs. Channel-specific workers render templates, apply preferences/quiet hours, and send via provider adapters with retries and exponential backoff. We design for at-least-once delivery with idempotent workers and a DLQ for poison messages. We monitor queue lag, provider success rates, and SLOs for critical notifications like OTP, and support provider failover during incidents.”

