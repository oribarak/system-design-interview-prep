# Uber / Ride-Sharing — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients
        RIDER[Rider App]
        DRIVER[Driver App]
    end

    subgraph API_Layer[API Layer]
        GW[API Gateway<br/>Auth, Rate Limit]
        WS[WebSocket Server<br/>Real-time tracking]
    end

    subgraph Core_Services[Core Services]
        RIDE_SVC[Ride Service<br/>Lifecycle State Machine]
        MATCH_SVC[Matching Service<br/>Driver Selection & Dispatch]
        LOC_SVC[Location Service<br/>Geospatial Index]
        PRICE_SVC[Pricing Service<br/>Fare + Surge]
        ETA_SVC[ETA Service<br/>Route + Traffic]
        NOTIF[Notification Service<br/>Push Notifications]
    end

    subgraph Location_Infra[Location Infrastructure]
        KAFKA_LOC[Kafka<br/>location-updates topic]
        GEO_IDX[In-Memory Geohash Index<br/>5M driver positions]
    end

    subgraph Pricing_Infra[Pricing Infrastructure]
        SURGE_COMP[Surge Computer<br/>Every 2 minutes]
        SURGE_CACHE[(Redis<br/>Surge multipliers per H3 cell)]
    end

    subgraph Storage[Storage]
        RIDE_DB[(PostgreSQL<br/>Rides, sharded by city)]
        USER_DB[(PostgreSQL<br/>Drivers & Riders)]
        TRIP_HIST[(Cassandra<br/>Trip History)]
        REDIS[(Redis<br/>Driver status, ride cache)]
    end

    subgraph External
        MAPS[Maps / Routing API]
        PAYMENT[Payment Service]
    end

    RIDER -->|request ride| GW
    DRIVER -->|location updates| GW
    RIDER <-->|tracking| WS
    DRIVER <-->|dispatch| WS

    GW --> RIDE_SVC
    GW --> PRICE_SVC
    RIDE_SVC -->|find driver| MATCH_SVC
    MATCH_SVC -->|query nearby| LOC_SVC
    MATCH_SVC -->|get ETA| ETA_SVC
    MATCH_SVC -->|dispatch| NOTIF
    NOTIF -->|push| DRIVER

    GW --> KAFKA_LOC
    KAFKA_LOC --> LOC_SVC
    LOC_SVC --> GEO_IDX

    PRICE_SVC --> SURGE_CACHE
    SURGE_COMP -->|compute| LOC_SVC
    SURGE_COMP -->|compute| RIDE_SVC
    SURGE_COMP -->|write| SURGE_CACHE

    ETA_SVC --> MAPS

    RIDE_SVC --> RIDE_DB
    RIDE_SVC --> REDIS
    MATCH_SVC --> REDIS
    RIDE_SVC --> TRIP_HIST
    RIDE_SVC --> PAYMENT

    LOC_SVC --> WS
```

## 2. Deep-Dive: Location Service and Geospatial Matching

```mermaid
flowchart TB
    subgraph Ingestion[Location Update Ingestion]
        DRIVER_APP[Driver App<br/>GPS every 3-5 sec]
        GW_IN[API Gateway]
        KAFKA[Kafka<br/>Partitioned by driver_id]
        CONSUMER[Kafka Consumer Group<br/>N consumers]
    end

    subgraph Geospatial_Index[In-Memory Geospatial Index]
        HASH_COMPUTE[Geohash Computer<br/>Precision 6: ~1.2km cells]
        CELL_MAP[Cell Hash Map<br/>geohash -> Set of DriverLocation]
        OLD_CELL[Remove from Old Cell<br/>If geohash changed]
        NEW_CELL[Add to New Cell]
        STALE[Stale Driver Cleaner<br/>Remove if no update > 30s]
    end

    subgraph Matching_Query[Proximity Query Flow]
        RIDER_LOC[Rider Pickup Location]
        RIDER_HASH[Compute Rider Geohash]
        NEIGHBORS[Query 9 Cells<br/>Center + 8 neighbors]
        FILTER_DIST[Haversine Distance Filter<br/>Within 5km radius]
        FILTER_STATUS[Status Filter<br/>AVAILABLE only]
        CANDIDATES[Candidate Drivers<br/>Typically 10-50]
    end

    subgraph Scoring[Driver Scoring]
        ETA_CALC[ETA Calculation<br/>Road distance, not straight line]
        RATING[Driver Rating<br/>4.0 - 5.0]
        ACCEPT_RATE[Acceptance Rate<br/>Historical]
        VEHICLE[Vehicle Type Match]
        SCORE[Composite Score<br/>0.4 ETA + 0.3 rating + 0.2 accept + 0.1 vehicle]
        RANKED[Ranked Driver List]
    end

    subgraph Dispatch[Sequential Dispatch]
        LOCK[Atomic Status Lock<br/>AVAILABLE -> DISPATCHED<br/>Redis CAS]
        OFFER[Send Ride Offer<br/>Push notification]
        TIMER[15-second Timer]
        ACCEPT{Driver<br/>accepts?}
        NEXT[Try Next Driver]
        MATCHED[Ride Matched!]
        UNLOCK[Unlock: DISPATCHED -> AVAILABLE]
    end

    DRIVER_APP --> GW_IN
    GW_IN --> KAFKA
    KAFKA --> CONSUMER
    CONSUMER --> HASH_COMPUTE
    HASH_COMPUTE --> OLD_CELL
    HASH_COMPUTE --> NEW_CELL
    OLD_CELL --> CELL_MAP
    NEW_CELL --> CELL_MAP
    STALE --> CELL_MAP

    RIDER_LOC --> RIDER_HASH
    RIDER_HASH --> NEIGHBORS
    NEIGHBORS --> CELL_MAP
    CELL_MAP --> FILTER_DIST
    FILTER_DIST --> FILTER_STATUS
    FILTER_STATUS --> CANDIDATES

    CANDIDATES --> ETA_CALC
    CANDIDATES --> RATING
    CANDIDATES --> ACCEPT_RATE
    CANDIDATES --> VEHICLE
    ETA_CALC --> SCORE
    RATING --> SCORE
    ACCEPT_RATE --> SCORE
    VEHICLE --> SCORE
    SCORE --> RANKED

    RANKED --> LOCK
    LOCK --> OFFER
    OFFER --> TIMER
    TIMER --> ACCEPT
    ACCEPT -->|yes| MATCHED
    ACCEPT -->|no/timeout| UNLOCK
    UNLOCK --> NEXT
    NEXT --> LOCK
```

## 3. Critical Path Sequence: Ride Request to Driver Matched

```mermaid
sequenceDiagram
    participant Rider as Rider App
    participant GW as API Gateway
    participant Ride as Ride Service
    participant Price as Pricing Service
    participant Surge as Surge Cache (Redis)
    participant Match as Matching Service
    participant Loc as Location Service
    participant ETA as ETA Service
    participant Redis as Redis (Driver Status)
    participant Notif as Notification Service
    participant Driver as Driver App
    participant DB as PostgreSQL

    Rider->>GW: POST /v1/rides/estimate {pickup, dropoff}
    GW->>Price: Compute fare estimate
    Price->>Surge: GET surge for H3 cell(pickup)
    Surge-->>Price: surge_multiplier = 1.3
    Price->>ETA: GET estimated trip duration
    ETA-->>Price: 25 minutes, 12 miles
    Price-->>GW: {fare: $25-32, eta: 4min, surge: 1.3}
    GW-->>Rider: Show estimate

    Rider->>GW: POST /v1/rides/request {pickup, dropoff, ride_type}
    GW->>Ride: Create ride
    Ride->>DB: INSERT ride (status=MATCHING)
    Ride->>Match: Find driver for ride_123

    Match->>Loc: Find nearby drivers (pickup, 5km, AVAILABLE)
    Loc->>Loc: Query geohash cells (center + 8 neighbors)
    Loc->>Loc: Filter by distance and status
    Loc-->>Match: 15 candidate drivers

    par Score candidates
        Match->>ETA: Batch ETA for 15 drivers to pickup
        ETA-->>Match: ETAs: [3min, 4min, 5min, ...]
    end

    Match->>Match: Score and rank candidates

    Note over Match,Driver: Sequential dispatch (try best driver first)

    Match->>Redis: SET driver_001 DISPATCHED (CAS, if AVAILABLE)
    Redis-->>Match: OK (locked)
    Match->>Notif: Send ride offer to driver_001
    Notif->>Driver: Push: "New ride, pickup 0.5 mi away"

    alt Driver accepts within 15s
        Driver->>GW: POST /rides/ride_123/accept
        GW->>Ride: Update ride status
        Ride->>DB: UPDATE ride SET status=ACCEPTED, driver_id=driver_001
        Ride->>Redis: SET driver_001 ON_TRIP
        Ride-->>GW: {status: ACCEPTED}
        GW->>Rider: Push: "Driver John is on the way, ETA 3 min"

        Note over Rider,Driver: Real-time tracking via WebSocket begins
    else Driver declines or timeout
        Match->>Redis: SET driver_001 AVAILABLE (unlock)
        Match->>Match: Try next ranked driver
    end
```
