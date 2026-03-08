# Uber / Ride-Sharing — Cheat Sheet

## Key Numbers
- 5M active drivers, 20M daily rides, 230 avg QPS (700 peak)
- 1.25M location updates/sec (5M drivers / 4s interval)
- In-memory geospatial index: ~250 MB for all driver positions
- Matching latency target: < 10 seconds
- Surge computation: every 2 minutes per H3 zone
- Trip storage: 40 GB/day

## Core Components
- **Location Service**: in-memory geohash index, ingests 1.25M updates/sec via Kafka
- **Matching Service**: scores nearby drivers (ETA 40%, rating 30%, acceptance 20%, vehicle 10%), sequential dispatch
- **Pricing Service**: base fare x surge multiplier, reads surge from Redis
- **Surge Computer**: batch job every 2 minutes, demand/supply ratio per H3 cell
- **Ride Service**: state machine (MATCHING -> ACCEPTED -> EN_ROUTE -> IN_PROGRESS -> COMPLETED)
- **WebSocket Server**: real-time driver location push to riders every 2-3 seconds

## Architecture in One Sentence
Driver GPS updates flow through Kafka into an in-memory geohash index, the matching service queries nearby available drivers, scores them, and sequentially dispatches with atomic Redis locking to prevent double-dispatch.

## Top 3 Trade-offs
1. **In-memory geohash vs PostGIS**: geohash is 10-100x faster but requires rebuild on crash (Kafka replay)
2. **Sequential vs broadcast dispatch**: sequential prevents double-accept but is slower by 15s/attempt
3. **H3 hexagonal vs square grid for surge**: H3 has uniform distances but adds library dependency

## Scaling Story
- 10x: shard location index by city/region, separate read/write paths
- 100x: partitioned geospatial index across nodes, predictive driver positioning, batched matching (Hungarian algorithm)

## Closing Statement
"Geohash-based in-memory index handles 1.25M location updates/sec. Matching scores drivers on ETA, rating, and acceptance within 10 seconds. Surge pricing computed every 2 minutes via H3 zones with temporal smoothing. Atomic Redis CAS prevents double-dispatch. Index recovers from Kafka replay on failure."
