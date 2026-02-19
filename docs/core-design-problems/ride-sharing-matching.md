# Design a Ride-sharing Matching System (high level) — full system design

Ride-sharing matching (Uber/Lyft-like) is a realtime geo-distributed problem:

- drivers send frequent location updates
- riders request trips
- system finds the best driver fast
- system must handle movement, cancellations, and fairness

This design focuses on the matching core and keeps mapping/route engines as integrations.

---

## 1) Requirements

### Functional requirements

- Drivers:
  - come online/offline
  - send location updates (every few seconds)
  - receive ride offers and accept/decline
- Riders:
  - request a ride (pickup, destination)
  - get matched to a driver
  - track driver arrival (ETA)
- Matching:
  - select a nearby available driver
  - handle timeouts, re-offers, cancellations

### Nice-to-have

- surge pricing
- pooling (shared rides)
- driver/rider ratings and preferences
- multi-stop trips

### Non-functional requirements

- Very low latency matching (often p95 < 1–2 seconds end-to-end)
- High write throughput (location updates)
- High availability (degrade gracefully)
- “Good enough” consistency for location (it’s inherently approximate)
- Fairness and anti-gaming (avoid always picking same drivers)

---

## 2) Scale estimation (example)

Assume:

- 1M drivers online at peak
- location update every 5 seconds

Updates/sec:

- 1M / 5 = 200k updates/sec

That’s a huge write workload.

Matching requests:

- maybe 5k–50k/sec depending on city/global scale

Key takeaway:

- location ingestion must be extremely write-optimized
- matching must be fast and avoid scanning large sets

---

## 3) High-level architecture

### Core components

- **Driver app / Rider app**
- **Location ingestion service**:
  - receives driver GPS updates
  - normalizes and stores “latest location”
- **Geo index / driver availability store**:
  - query: “available drivers near (lat, lon)”
- **Matching service**:
  - selects candidates
  - ranks by ETA and rules
  - sends offers and handles accept/decline
- **Trip service**:
  - trip lifecycle state machine (requested, matched, started, completed)
- **Dispatch/notification service**:
  - pushes offers to driver devices
- **Maps/ETA service integration**:
  - external or internal routing engine

---

## 4) Data model and storage choices

### Driver state (ephemeral)

You need “latest location + availability”:

- driver_id
- lat/lon
- updated_at
- status: available / on_trip / offline

Because updates are frequent and data is ephemeral:

- store in an in-memory KV (Redis) or a specialized geo store
- use TTL to remove stale drivers automatically

### Trips (durable)

Trip records must be durable:

- trip_id
- rider_id
- driver_id (once matched)
- pickup/destination
- status
- timestamps

Store in SQL (transactions and correctness matter).

---

## 5) Geo indexing (how to find nearby drivers fast)

You cannot scan all drivers.

Common techniques:

### A) Geohash grid

Convert (lat, lon) into a geohash string representing a grid cell.

- nearby points share prefixes
- you can query “drivers in this cell and neighboring cells”

### B) S2 cells (Google S2)

Another popular hierarchical cell system.

### Query strategy

1. Compute pickup cell (at a chosen resolution).
2. Fetch available drivers in that cell.
3. If not enough, expand to neighboring cells (ring expansion).

Key tradeoff:

- smaller cells: fewer candidates per cell, more neighbor queries
- larger cells: more candidates, more filtering cost

---

## 6) Matching algorithm (practical)

High-level steps:

1. Rider requests trip.
2. Matching service finds candidate drivers near pickup:
   - filter by availability and basic constraints
3. Compute a score for each candidate:
   - estimated pickup ETA
   - driver acceptance rate / fairness weights
   - distance to pickup
4. Offer to best driver (or top N sequentially).
5. If driver accepts within timeout → assign trip.
6. If driver declines/timeouts → offer next candidate.

Why not offer to all at once?

- many drivers accept; you must resolve conflict
- creates a bad driver experience

Some systems do “batch offers”:

- offer to a small set with ranking and stop after first accept

---

## 7) Trip lifecycle and race conditions

Two common race conditions:

- two drivers accept the same trip
- one driver is matched to two trips

Solution:

- trip assignment must be atomic.

Approach:

- Trip service owns the truth.
- Matching service requests “claim trip” with conditional update:
  - “set driver_id if status is requested”
- Driver availability is also updated atomically:
  - “set driver status to on_trip if currently available”

Use:

- DB transaction if in one DB
- or compare-and-swap/version checks if distributed

---

## 8) Handling movement (staleness is normal)

GPS updates are noisy and delayed.

Design choices:

- treat location as approximate
- store only latest location per driver (not full history) for matching
- use TTL to discard stale drivers (if no update in 30–60s, mark unavailable)

---

## 9) Hotspots (downtown problem)

Downtown cells are hot:

- many drivers + many riders
- queries and updates concentrate

Mitigations:

- dynamic cell sizing:
  - split hot cells into smaller ones
- sharding by `(cell_id, shard)` for large cells
- partition services by geography (city/region)
- caching candidate lists briefly (seconds)

---

## 10) Reliability and failure modes

### Location store down

- matching degrades or pauses
- fall back to last known locations from a slower store (if available)

### Dispatch failures

- driver offer might not be delivered
- use retries and alternative channels (push + websocket)

### Partial outages

- isolate by region/city so one city doesn’t impact another

---

## 11) Security and abuse

- validate driver location updates (rate limit, sanity checks)
- prevent spoofing (device attestation signals)
- protect rider phone/location PII

---

## 12) Observability

Metrics:

- matching latency (request → assigned)
- offer acceptance rate
- cancellation rate
- location update ingest rate and lag
- hot cells (top cell QPS)

---

## 13) Interview-ready summary (30 seconds)

“Drivers stream frequent location updates to a write-optimized ingestion service that stores each driver’s latest location and availability in an in-memory geo-indexed store (geohash/S2 cells) with TTLs. When a rider requests a trip, the matching service queries nearby cells to get candidate drivers, ranks them by ETA and fairness rules, and sends offers with timeouts. Trip assignment is made atomic via conditional updates so a driver/trip can’t be double-assigned. We handle hotspots by dynamically splitting busy geo cells and sharding by geography, and we monitor matching latency, acceptance rates, and hot cell load.”

