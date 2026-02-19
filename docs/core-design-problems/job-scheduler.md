# Design a Job Scheduler — full system design

A job scheduler runs tasks:

- **now** (background jobs)
- **later** (delayed jobs)
- **on a schedule** (cron)

Schedulers are everywhere:

- send “reminder” emails at a later time
- run nightly billing jobs
- run data backfills
- trigger retries

The core challenge is reliability:

> Jobs must run at least once, even if workers crash.

---

## 1) Requirements

### Functional requirements

- Create jobs:
  - run immediately
  - run at a specific time (delayed)
  - run on a cron schedule (e.g., every hour)
- Job configuration:
  - payload (what to do)
  - retry policy
  - timeout
  - concurrency limits
- Job execution:
  - workers pick jobs and execute
  - report success/failure
- Visibility:
  - job status (pending/running/succeeded/failed)
  - job history and attempts

### Non-functional requirements

- At-least-once execution (standard)
- Scalable throughput
- Operational tooling (search jobs, retry, cancel)
- Multi-tenant fairness (optional)

### Out of scope (unless asked)

- Exactly-once execution (usually too hard; use idempotency instead)
- Complex workflow orchestration (DAGs) (can be a follow-up)

---

## 2) High-level architecture

### Components

- **Job API**: create/cancel/query jobs
- **Job store**: durable DB for job definitions and state
- **Scheduler**: finds due jobs and enqueues them
- **Queue**: holds jobs to execute
- **Workers**: execute jobs, report results
- **Lease/lock mechanism**: prevents multiple workers from executing the same job
- **DLQ**: for repeatedly failing jobs

Two common designs:

1. DB as source of truth + queue for execution (most common)
2. Queue as primary with delayed features (works for some systems)

---

## 3) Data model (simple)

### jobs table

- `job_id` (PK)
- `type` (email_send, report_generate, ...)
- `payload` (JSON)
- `run_at` (timestamp; now for immediate jobs)
- `status` (pending, enqueued, running, succeeded, failed, cancelled)
- `attempt` (int)
- `max_attempts`
- `timeout_sec`
- `created_at`, `updated_at`

### job_attempts table (optional but helpful)

- `job_id`
- `attempt_no`
- `started_at`, `finished_at`
- `status`
- `error`
- `worker_id`

Index:

- `(status, run_at)` so scheduler can find due jobs efficiently.

---

## 4) Scheduling due jobs

### The scheduler loop

Every small interval (e.g., 1s–10s):

1. query jobs:
   - `status = pending`
   - `run_at <= now`
   - limit batch size (e.g., 1k)
2. for each job:
   - mark as `enqueued` (atomically)
   - publish job to queue

Important:

- step 2 must be safe under concurrency (multiple scheduler instances).

How:

- use an atomic update:
  - “update job set status=enqueued where job_id=? and status=pending”
- if update affects 0 rows, another scheduler already took it

This is the simplest “distributed lock” pattern for schedulers.

---

## 5) Worker execution model

Worker steps:

1. consume job from queue
2. acquire a lease (optional if queue guarantees single delivery; but queue is often at-least-once)
3. mark job `running` with `lease_expires_at`
4. execute with timeout
5. on success:
   - mark `succeeded`
6. on failure:
   - increment attempt
   - if attempts left → reschedule with backoff
   - else → mark `failed` and/or send to DLQ

---

## 6) Retries and backoff

Retry is essential but can cause storms if uncontrolled.

Common policy:

- exponential backoff + jitter
- cap max delay

Example:

- 1m, 5m, 30m, 2h, then fail

Store next run time:

- set `run_at = now + backoff`
- set `status = pending`

---

## 7) Idempotency (job handlers must be safe)

Because of at-least-once delivery, jobs can run twice.

Examples:

- “send email” can be duplicated
- “charge customer” must never duplicate

Fix:

- design job handlers to be idempotent:
  - use unique keys (e.g., `email_send_id`)
  - record side effects in DB with uniqueness constraints

Senior signal:

- “We prefer at-least-once execution plus idempotent handlers over exactly-once.”

---

## 8) Concurrency limits

You may need:

- global concurrency for a job type
- per-tenant concurrency
- per-resource concurrency (e.g., one job per user at a time)

Implementation:

- semaphore counters in Redis with TTL/leases
- or DB row locks (careful with contention)

---

## 9) Cron jobs

Cron schedules generate jobs repeatedly.

Model:

- store a cron definition (e.g., `0 * * * *`)
- scheduler calculates next run time after each run

Prevent duplicates:

- use a unique key `(cron_id, scheduled_time)` to ensure only one instance is created.

---

## 10) Failure modes

### Worker crashes mid-job

Lease expires → job becomes eligible to retry.

### Scheduler crashes

Multiple schedulers run; atomic “claim” prevents duplicates.

### Queue delayed or down

Jobs remain in DB; scheduler can re-enqueue when queue recovers.

### Poison pill jobs

Send to DLQ after N failures; alert humans.

---

## 11) Observability and operations

Metrics:

- queue depth/lag
- job throughput (created/succeeded/failed)
- time-to-start (delay from run_at to actual start)
- retry rate
- DLQ size

Operational tools:

- search jobs by type/status
- retry/cancel manually
- view attempts and error logs

---

## 12) Interview-ready summary (30 seconds)

“We store jobs durably in a DB with `run_at` and `status`, and run a scheduler loop that atomically claims due jobs and enqueues them to a worker queue. Workers execute jobs with leases/timeouts and record attempts, using at-least-once delivery plus idempotent handlers. Failures are retried with exponential backoff; poison jobs go to a DLQ. We enforce concurrency limits per job type/tenant and monitor queue lag, job delay, retry rates, and DLQ growth.”

