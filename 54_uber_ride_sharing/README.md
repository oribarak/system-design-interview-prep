# System Design Interview: Uber / Ride-Sharing — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a ride-sharing platform like Uber that connects riders with nearby drivers in real-time. The system handles ride requests, driver matching, real-time location tracking, dynamic pricing, ETA computation, and trip management. I'll focus on the real-time location service for tracking millions of drivers, the matching algorithm that pairs riders with optimal drivers, and how we handle the high-throughput geospatial queries that make this all work at scale."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | How many active drivers and riders? | 5M active drivers, 20M daily riders |
| 2 | Geographic scope? | Global, 500+ cities |
| 3 | Ride types? Solo, pool, premium? | All: UberX, Pool, Premium, XL |
| 4 | Driver location update frequency? | Every 3-5 seconds while online |
| 5 | Matching radius? | Typically 5 km, up to 15 km in sparse areas |
| 6 | How fast should matching be? | < 10 seconds from request to driver assignment |
| 7 | Dynamic pricing (surge)? | Yes, based on supply/demand ratio per zone |
| 8 | Payment handling? | Yes, but can treat payment service as external |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Active drivers (online) | 5M globally, ~1M at peak in a region |
| Location updates/sec | 5M drivers / 4 sec avg = 1.25M updates/sec |
| Ride requests/day | 20M rides/day |
| Ride request QPS | 20M / 86,400 ~ 230 QPS avg, ~700 QPS peak |
| Matching queries/sec | 700 (each queries nearby drivers) |
| Location data per update | driver_id + lat/lng + timestamp ~ 50 bytes |
| Location update bandwidth | 1.25M * 50 bytes = 62.5 MB/sec |
| Location storage (in-memory) | 5M drivers * 50 bytes = 250 MB (fits in RAM) |
| Trip records/day | 20M * 2 KB = 40 GB/day |
| ETA computation/sec | ~5K (ride requests + rider app checks) |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Rider request**: rider submits pickup/dropoff, gets price estimate and ETA
2. **Driver matching**: system finds optimal nearby driver and dispatches the ride
3. **Real-time tracking**: rider sees driver approach on map; driver sees navigation
4. **Trip lifecycle**: request -> match -> pickup -> in-progress -> dropoff -> payment
5. **Dynamic pricing**: surge pricing based on supply/demand in geographic zones
6. **Driver management**: drivers go online/offline, accept/decline rides

### Non-Functional Requirements
- **Matching latency**: < 10 seconds from request to driver assignment
- **Location freshness**: driver position accurate within 5 seconds
- **Availability**: 99.99% for ride requests and matching
- **Scalability**: 5M concurrent drivers, 20M daily rides
- **Consistency**: exactly-one matching (no double-dispatching a driver)
- **Global**: multi-region deployment across 500+ cities

## 4) API Design (2-4 min)

### Rider APIs
```
POST /v1/rides/estimate
{ "pickup": {"lat": 37.7749, "lng": -122.4194}, "dropoff": {"lat": 37.3382, "lng": -121.8863}, "ride_type": "UberX" }
Response: { "estimated_fare": "$25-32", "eta_minutes": 4, "surge_multiplier": 1.3 }

POST /v1/rides/request
{ "pickup": {...}, "dropoff": {...}, "ride_type": "UberX", "payment_method_id": "pm_abc" }
Response: { "ride_id": "ride_123", "status": "MATCHING", "eta_minutes": 4 }

GET /v1/rides/{ride_id}/status
Response: { "status": "DRIVER_EN_ROUTE", "driver": {...}, "driver_location": {...}, "eta_minutes": 3 }
```

### Driver APIs
```
POST /v1/drivers/location
{ "lat": 37.7750, "lng": -122.4195, "heading": 270, "speed": 25 }

PUT /v1/drivers/status
{ "status": "ONLINE" | "OFFLINE" | "ON_TRIP" }

POST /v1/rides/{ride_id}/accept
POST /v1/rides/{ride_id}/arrive
POST /v1/rides/{ride_id}/start
POST /v1/rides/{ride_id}/complete
```

### WebSocket (real-time)
```
ws://api.uber.com/v1/tracking/{ride_id}
// Pushes driver location updates every 2-3 seconds to rider
```

## 5) Data Model (3-5 min)

### Core Entities

**Driver**
| Column | Type | Notes |
|--------|------|-------|
| driver_id | UUID | PK |
| name | string | |
| vehicle_type | enum | UberX, Premium, XL |
| status | enum | OFFLINE, AVAILABLE, ON_TRIP |
| rating | float | |
| city_id | string | |

**DriverLocation** (in-memory, real-time)
| Column | Type | Notes |
|--------|------|-------|
| driver_id | UUID | PK |
| lat | double | |
| lng | double | |
| heading | int | degrees |
| speed | float | km/h |
| updated_at | timestamp | |

**Ride**
| Column | Type | Notes |
|--------|------|-------|
| ride_id | UUID | PK |
| rider_id | UUID | |
| driver_id | UUID | nullable until matched |
| pickup_location | point | lat/lng |
| dropoff_location | point | lat/lng |
| status | enum | MATCHING, ACCEPTED, EN_ROUTE, PICKUP, IN_PROGRESS, COMPLETED, CANCELLED |
| fare_estimate | decimal | |
| actual_fare | decimal | |
| surge_multiplier | float | |
| created_at | timestamp | |

### Storage Choices
- **Driver locations**: in-memory geospatial index (Geohash-based or QuadTree) -- fast spatial queries
- **Rides**: PostgreSQL with sharding by city_id -- relational, ACID for ride state transitions
- **Trip history**: Cassandra -- append-only, high write throughput, partitioned by rider_id
- **Driver/Rider profiles**: PostgreSQL -- relational
- **Real-time events**: Kafka -- driver location updates, ride state changes
- **Caching**: Redis -- ride status, driver status, surge pricing

## 6) High-Level Architecture (5-8 min)

When a rider requests a ride, the Ride Service creates a ride record and asks the Matching Service to find nearby available drivers. The Matching Service queries the Location Service (which maintains an in-memory geospatial index of all driver positions), scores candidate drivers, and dispatches the best match. The driver accepts, and real-time tracking begins via WebSocket.

### Components
- **API Gateway**: auth, rate limiting, WebSocket upgrade
- **Ride Service**: manages ride lifecycle (state machine), fare calculation
- **Matching Service**: finds and dispatches optimal driver for a ride request
- **Location Service**: ingests driver location updates, maintains geospatial index, answers proximity queries
- **Pricing Service**: computes fare estimates, manages surge pricing per zone
- **ETA Service**: computes estimated arrival times using road graph + traffic data
- **Notification Service**: push notifications to drivers (new ride offer) and riders (driver assigned)
- **Payment Service**: handles charging and driver payouts (external integration)
- **Trip History Service**: stores completed trip records

## 7) Deep Dive #1: Location Service and Geospatial Matching (8-12 min)

### Geospatial Index Design

**Approach: Geohash-based in-memory index**

1. Divide the world into geohash cells (precision level 6: ~1.2 km x 0.6 km cells)
2. Maintain a hash map: `geohash_cell -> Set<DriverLocation>`
3. When a driver sends a location update:
   - Compute new geohash
   - If geohash changed: remove from old cell, add to new cell
   - Update location in the set
4. When finding nearby drivers:
   - Compute geohash of rider's location
   - Query the rider's cell + 8 surrounding cells (covers ~3.6 km radius)
   - Filter by Haversine distance for exact radius
   - Filter by driver status (AVAILABLE only)

**Why geohash over QuadTree?**
- Geohash maps to string prefixes, easy to shard by prefix
- Consistent cell boundaries across nodes
- Simple to implement and reason about

### Location Update Pipeline

```
Driver App -> API Gateway -> Kafka (location-updates topic) -> Location Service consumers
```

- Kafka partitioned by `driver_id % num_partitions` for ordered updates per driver
- Location Service consumers update the in-memory geospatial index
- Stale location cleanup: if no update in 30 seconds, mark driver as stale

### Matching Algorithm

```python
def match_ride(ride_request):
    # 1. Find nearby available drivers (< 5 km)
    candidates = location_service.find_nearby(
        ride_request.pickup, radius_km=5, status=AVAILABLE
    )

    if not candidates:
        # Expand radius
        candidates = location_service.find_nearby(
            ride_request.pickup, radius_km=15, status=AVAILABLE
        )

    # 2. Score candidates
    for driver in candidates:
        driver.score = (
            0.4 * normalize(driver.eta_to_pickup) +  # Lower ETA is better
            0.3 * normalize(driver.rating) +           # Higher rating is better
            0.2 * normalize(driver.acceptance_rate) +   # Higher acceptance is better
            0.1 * vehicle_type_match(driver, ride_request)
        )

    # 3. Sort by score, offer to best driver
    candidates.sort(key=lambda d: d.score, reverse=True)

    # 4. Sequential dispatch with timeout
    for driver in candidates[:5]:
        offer = dispatch_to_driver(driver, ride_request, timeout=15s)
        if offer.accepted:
            return driver

    # 5. No driver accepted
    return NO_MATCH
```

### Preventing Double-Dispatch
- When a driver is offered a ride, set their status to `DISPATCHED` atomically (Redis CAS or DB row lock)
- A `DISPATCHED` driver is excluded from other matching queries
- If the driver declines or timeout expires, revert to `AVAILABLE`
- Use distributed lock with TTL to prevent stale locks

## 8) Deep Dive #2: Dynamic Pricing (Surge) (5-8 min)

### Zone-Based Supply/Demand

1. **Grid the city**: divide into hexagonal zones (H3 cells, resolution 7: ~5 km2)
2. **Compute supply**: count AVAILABLE drivers in each zone (from Location Service)
3. **Compute demand**: count ride requests in each zone in the last 5 minutes
4. **Surge multiplier**: `surge = max(1.0, demand / (supply * capacity_factor))`

### Surge Computation Pipeline
- Every 2 minutes, a batch job:
  1. Queries Location Service for driver counts per H3 cell
  2. Queries Ride Service for request counts per H3 cell (last 5 min)
  3. Computes surge multiplier per cell
  4. Writes to Redis (surge cache) with 2-minute TTL
- Pricing Service reads surge from Redis when computing fare estimates

### Smoothing and Caps
- **Temporal smoothing**: exponential moving average over last 3 computation windows
- **Spatial smoothing**: average surge with neighboring cells to avoid sharp boundaries
- **Cap**: maximum surge multiplier (e.g., 5x) to prevent price gouging
- **Ramp-up/ramp-down**: surge changes by at most 0.5x per 2-minute window

### Fare Calculation
```
base_fare = base_rate + (per_mile_rate * distance) + (per_minute_rate * duration)
fare = base_fare * surge_multiplier
fare = max(fare, minimum_fare)
fare = min(fare, maximum_fare_cap)
```

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| Geohash index (in-memory) | PostGIS spatial queries | In-memory is 10-100x faster for proximity queries at this QPS |
| Sequential driver dispatch | Broadcast to multiple drivers | Sequential avoids multiple drivers accepting same ride; broadcast is faster but complex |
| Kafka for location updates | Direct writes to Location Service | Kafka buffers spikes, provides durability, enables multiple consumers |
| H3 hexagonal grid for surge | Square grid / geohash | H3 has uniform distance from center to edge; square grids have corner distortion |
| Redis for surge cache | Compute on every request | Pre-computed surge avoids per-request computation cost |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (50M drivers, 200M rides/day)
- **Shard Location Service by city/region**: each city has its own geospatial index
- **Separate read/write paths**: location updates write to Kafka; matching reads from in-memory index
- **Regional deployment**: matching only queries drivers in the same metro area
- **Async fare finalization**: compute exact fare after trip, not during

### 100x Scale (500M drivers, 2B rides/day)
- **Partitioned geospatial index**: shard geohash cells across nodes, route queries by location
- **Predictive matching**: pre-position drivers before demand materializes using ML demand forecasting
- **Edge location processing**: aggregate location updates at edge before forwarding to central
- **Batched matching**: group ride requests in same area, solve assignment as optimization problem (Hungarian algorithm)

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| Location Service node crash | In-memory index rebuilt from Kafka replay (last 30 seconds of updates) |
| Matching Service down | Ride requests queued in Kafka; match when service recovers |
| Driver loses connectivity | Grace period (30s); if reconnects, resume. Otherwise, cancel and re-match. |
| Double-dispatch (race condition) | Atomic status check-and-set in Redis with TTL |
| Payment failure | Complete trip, queue payment retry; don't block ride completion |
| Kafka lag on location updates | Stale driver positions; matching uses last known position with confidence decay |

### Ride State Machine Consistency
- Ride status transitions enforced via database: `UPDATE rides SET status = 'ACCEPTED' WHERE ride_id = ? AND status = 'MATCHING'`
- Only one transition succeeds (optimistic concurrency)
- All state changes published to Kafka for downstream consumers

## 12) Observability and Operations (2-4 min)

### Key Metrics
- Matching time p50/p99 (target: < 10s)
- Match success rate (% of requests that find a driver)
- Driver utilization (% of online time spent on trips)
- Surge multiplier distribution per city
- Location update lag (time from driver GPS to index update)
- ETA accuracy (predicted vs actual pickup time)

### Alerts
- Match success rate drops below 80% in any city
- Location update lag exceeds 10 seconds
- Surge multiplier exceeds 4x in any zone for > 10 minutes
- Ride cancellation rate spikes above 15%

## 13) Security (1-3 min)

- **Location privacy**: driver locations not exposed to riders until matched; rider location not exposed to unmatched drivers
- **Trip data encryption**: all location data encrypted at rest and in transit
- **Authentication**: driver and rider apps use OAuth 2.0 with device-specific tokens
- **Rate limiting**: prevent location spoofing by validating GPS delta between updates (speed check)
- **Payment security**: PCI-DSS compliant payment processing via external provider
- **Safety features**: trip sharing with emergency contacts, panic button integrations

## 14) Team and Operational Considerations (1-2 min)

- **Location/Maps team (5-8 engineers)**: geospatial index, ETA computation, routing
- **Matching team (4-6 engineers)**: matching algorithm, dispatch, supply positioning
- **Pricing team (3-4 engineers)**: surge pricing, fare computation, promotions
- **Platform team (4-6 engineers)**: API gateway, ride lifecycle, real-time streaming
- **On-call**: city-level monitoring, per-city escalation for matching failures

## 15) Common Follow-up Questions

**Q: How does ride pooling (shared rides) work?**
A: Pooling matching is a combinatorial optimization problem. When a pool request comes in, check existing in-progress pool trips that can detour to the new pickup with < 5 minutes added. Use dynamic programming to optimize the multi-stop route. Fare is split based on solo-equivalent pricing.

**Q: How do you handle driver positioning (directing drivers to high-demand areas)?**
A: Demand prediction model (time-of-day, events, weather) forecasts demand per zone 15-30 minutes ahead. Suggest repositioning to drivers in low-demand zones via app notifications. Incentivize with bonuses for being in target zones.

**Q: How accurate are ETA predictions?**
A: ETA uses road graph (OpenStreetMap) + real-time traffic data (from driver GPS traces) + ML model trained on historical trip durations. Accuracy is typically within 2 minutes for < 15-minute ETAs.

**Q: How do you handle cancellations?**
A: Rider cancellation after driver assigned: cancellation fee (if > 2 minutes after acceptance). Driver cancellation: penalizes acceptance rate, ride re-enters matching queue. Mutual cancellation: no fee.

**Q: How do you handle cross-city or airport rides?**
A: Pricing uses actual route distance/duration, not zone-based. Driver can accept/decline long trips. Airport pickups use geofenced queue systems.

## 16) Closing Summary (30-60s)

"I've designed a ride-sharing system handling 5M concurrent drivers and 20M daily rides. The Location Service maintains an in-memory geohash index of all driver positions, updated via Kafka at 1.25M events/sec. The Matching Service scores nearby drivers on ETA, rating, and acceptance rate, dispatching sequentially with atomic status locking to prevent double-dispatch. Dynamic pricing uses H3 hexagonal zones with 2-minute computation cycles, smoothed temporally and spatially. The system scales by sharding the geospatial index per city/region, and recovers from Location Service failures by replaying recent Kafka events."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| geohash_precision | 6 | 5 - 8 | Lower = larger cells, fewer to query; higher = smaller cells, more precise |
| matching_radius_km | 5 | 2 - 15 | Larger radius = more candidates but longer ETAs |
| dispatch_timeout_sec | 15 | 10 - 30 | Time driver has to accept before moving to next |
| max_dispatch_attempts | 5 | 3 - 10 | Number of drivers to try before failing |
| surge_computation_interval_sec | 120 | 60 - 300 | More frequent = more responsive pricing |
| max_surge_multiplier | 5.0 | 2.0 - 10.0 | Cap on surge pricing |
| location_staleness_threshold_sec | 30 | 15 - 60 | Time after which driver marked stale |
| driver_location_update_interval_sec | 4 | 2 - 10 | More frequent = more accurate, more bandwidth |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| Geohash | Hierarchical spatial encoding that maps lat/lng to a string prefix |
| H3 | Uber's hexagonal hierarchical spatial index system |
| Haversine | Formula for calculating great-circle distance between two lat/lng points |
| Surge Pricing | Dynamic price multiplier based on supply/demand ratio |
| Dispatch | The act of offering a ride to a specific driver |
| Double-Dispatch | Bug where two rides are offered to the same driver simultaneously |
| ETA | Estimated Time of Arrival |
| Geofence | Virtual boundary around a geographic area (e.g., airport pickup zone) |
| Supply Positioning | Directing drivers to areas of predicted high demand |
| Hungarian Algorithm | Optimal assignment algorithm for matching riders to drivers in batch |

## Appendix C: References

- Uber Engineering: "Scaling Uber's Real-Time Market Platform"
- H3: Uber's hexagonal hierarchical spatial index
- Google S2 Geometry Library
- "Designing Data-Intensive Applications" by Martin Kleppmann
- Uber Engineering: "How Uber Manages a Million Writes Per Second Using Mesos and Cassandra"
- Lyft Engineering: "Matchmaking in Lyft Line"
