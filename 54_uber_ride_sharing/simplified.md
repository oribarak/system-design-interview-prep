# Uber / Ride-Sharing System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a ride-sharing platform that connects riders with nearby drivers in real-time across hundreds of cities. The hard part is maintaining an in-memory geospatial index of millions of driver positions updated every few seconds, running a matching algorithm that pairs riders with optimal drivers in under 10 seconds, and computing dynamic surge pricing based on real-time supply and demand.

## 2. Requirements
**Functional:** Riders submit pickup/dropoff and get price estimates and ETAs. System finds and dispatches the optimal nearby driver. Real-time location tracking during the ride. Dynamic surge pricing based on supply/demand per zone. Trip lifecycle management from request through payment.

**Non-functional:** Matching in under 10 seconds. Driver positions accurate within 5 seconds. 99.99% availability for ride requests. Support 5M concurrent drivers and 20M daily rides. Exactly-one matching -- no double-dispatching a driver.

## 3. Core Concept: Geohash-Based In-Memory Spatial Index
The Location Service maintains a hash map of geohash cells to sets of driver locations, entirely in memory (5M drivers at 50 bytes each is only 250MB). When a driver sends a GPS update, we compute the new geohash, move them between cells if needed. When matching a rider, we query the rider's cell plus 8 surrounding cells (covering roughly 3.6km radius), filter by Haversine distance, and only consider AVAILABLE drivers. This gives us O(1) proximity lookups instead of scanning all drivers.

## 4. High-Level Architecture
```
Rider App --> [API Gateway] --> [Ride Service] --> [Matching Service]
                                                        |
Driver App --> [Kafka] --> [Location Service] -- in-memory geohash index
                                                        |
                                [Pricing Service] -- surge from H3 zones
                                [ETA Service] -- road graph + traffic
```
- **Location Service**: Ingests 1.25M location updates/sec via Kafka, maintains in-memory geohash index.
- **Matching Service**: Queries nearby drivers, scores by ETA/rating/acceptance rate, dispatches sequentially.
- **Ride Service**: Manages ride lifecycle state machine with optimistic concurrency in the database.
- **Pricing Service**: Computes surge multipliers per H3 hexagonal zone every 2 minutes.

## 5. How a Ride Gets Matched
1. Rider submits pickup/dropoff; Pricing Service calculates fare with current surge multiplier.
2. Ride Service creates a ride record with status MATCHING.
3. Matching Service queries Location Service for available drivers within 5km (expand to 15km if sparse).
4. Candidates are scored: 40% pickup ETA, 30% rating, 20% acceptance rate, 10% vehicle match.
5. Best driver is offered the ride; their status is atomically set to DISPATCHED (prevents double-dispatch).
6. Driver has 15 seconds to accept; if declined, next candidate is tried (up to 5 attempts).
7. On acceptance, ride status transitions to ACCEPTED and real-time tracking begins via WebSocket.

## 6. What Happens When Things Fail?
- **Location Service node crash**: In-memory index rebuilt from Kafka replay of the last 30 seconds of location updates.
- **Double-dispatch race condition**: Atomic status check-and-set in Redis with TTL. Only one ride can claim a driver.
- **Driver loses connectivity**: 30-second grace period. If reconnection fails, cancel and re-match the rider.

## 7. Scaling
- **10x**: Shard Location Service by city/region. Each city gets its own geospatial index. Separate read/write paths for location updates vs matching queries.
- **100x**: Partition geohash cells across nodes. Predictive matching using ML demand forecasting to pre-position drivers. Batched matching using the Hungarian algorithm to solve assignment as an optimization problem.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Geohash in-memory index vs PostGIS | In-memory is 10-100x faster for proximity queries at this QPS; PostGIS is simpler but too slow |
| Sequential driver dispatch vs broadcast | Sequential prevents multiple accepts for the same ride; broadcast is faster but adds complexity |
| H3 hexagonal zones for surge | Uniform distance from center to edge avoids corner distortion of square grids |

## 9. Closing (30s)
> The system centers on an in-memory geohash index that holds all driver positions (250MB total) and supports 1.25M updates/sec via Kafka. Matching scores nearby drivers by ETA, rating, and acceptance rate, dispatching sequentially with atomic status locking to prevent double-dispatch. Surge pricing uses H3 hexagonal zones recomputed every 2 minutes. The system scales by sharding the geospatial index per city and recovers from Location Service failures by replaying recent Kafka events.
