# Food Delivery System -- Cheat Sheet

## Key Numbers
- 10M orders/day, ~556 peak OPS, 500K concurrent active orders
- 200K concurrent drivers, 40K location updates/sec
- 500K restaurants, 50M menu items
- p50 order placement < 1s, tracking latency < 5-10s

## Core Components
- Restaurant Service: search via Elasticsearch, menu browsing, ratings
- Order Service: state machine (PLACED->CONFIRMED->PREPARING->READY->PICKED_UP->DELIVERED), PostgreSQL sharded by customer_id
- Dispatch Service: batch-based driver-order matching every 30s using Redis GEOSEARCH + scoring + bipartite matching
- Driver Location Service: ingests 40K GPS updates/sec into Redis geospatial index
- Tracking Service: pushes driver location to customers via WebSocket
- ETA Service: ML model predicting prep time + travel time
- Payment Service: authorize at placement, capture at delivery

## Architecture in One Sentence
Customers search restaurants via Elasticsearch, place orders through a strongly consistent order service, a batch dispatch engine matches drivers using Redis geospatial proximity, and WebSocket-based tracking streams live driver locations to customers.

## Top 3 Trade-offs
1. Batch dispatch (30s window) vs. greedy nearest-driver: batch yields globally better assignments at the cost of slight dispatch delay.
2. Redis geospatial vs. PostGIS for driver locations: Redis wins on write throughput (40K/s) and read latency for radius queries.
3. WebSocket vs. polling for tracking: WebSocket reduces bandwidth and latency but requires connection state management at scale.

## Scaling Story
10x: shard by city/region, per-city Redis clusters, Elasticsearch read replicas, horizontal WebSocket servers.
100x: multi-region with data sovereignty, ML-based dispatch (reinforcement learning), custom geospatial store, pre-computed ETAs.

## Closing Statement
A food delivery platform combining Elasticsearch-based restaurant discovery, a strongly consistent order service, batch-optimized dispatch via Redis geospatial queries, and WebSocket real-time tracking handles 10M daily orders with sub-second placement and 5-second tracking freshness.
