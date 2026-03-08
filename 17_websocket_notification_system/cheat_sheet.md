# WebSocket Notification System -- Cheat Sheet

## Key Numbers
- 50M concurrent WebSocket connections, 500 gateway servers (100K conn/server)
- 500K notifications/sec peak, 5B notifications/day
- p99 delivery latency < 500ms, notification payload ~500 bytes
- 5 TB storage/day, 90-day retention (450 TB), Cassandra with TTL
- 100 GB total connection state in Redis (~200 MB/server)

## Core Components (one line each)
- **WebSocket Gateway**: Stateful edge servers holding persistent connections, 100K each
- **Connection Registry (Redis)**: Maps user_id to gateway server(s) for O(1) routing
- **Notification Router**: Stateless workers consuming Kafka, applying preferences, routing via Redis pub/sub
- **Kafka**: Durable event bus partitioned by user_id for ordering guarantees
- **Aggregation Service**: Collapses similar notifications using collapse_key with 5s tumbling window
- **Cassandra**: Notification store partitioned by user_id, TIMEUUID clustering for time-ordered reads
- **Preference Service**: Filters muted notification types, backed by PostgreSQL

## Architecture in One Sentence
Producers emit events to Kafka (partitioned by user_id), stateless routers apply preferences and aggregation then route to the correct WebSocket gateway via Redis pub/sub lookup, and gateways push notifications down persistent WSS connections with ACK-based guaranteed delivery.

## Top 3 Trade-offs
1. **WebSocket vs SSE/polling**: WebSocket chosen for bidirectional communication (ACKs, subscriptions) and lowest per-message overhead despite stateful complexity
2. **Fan-out on write vs read**: Write-path fan-out for real-time targeted notifications; read-path fan-out for bulk broadcasts to reduce write amplification
3. **Redis pub/sub (fire-and-forget) for gateway routing**: Acceptable because guaranteed delivery is handled by Cassandra persistence + reconnect replay

## Scaling Story
- **10x (500M connections)**: 5K gateway servers across regions, more Kafka partitions, aggressive aggregation
- **100x (5B connections)**: Edge PoPs worldwide, hierarchical fan-out tree, custom binary protocol (QUIC), tiered storage (Redis/Cassandra/S3)

## Closing Statement
A two-tier fan-out architecture (Kafka for user-level, Redis pub/sub for server-level) with persistent connection registry, 5-second aggregation windows, and ACK-based guaranteed delivery provides sub-500ms real-time notifications at 50M concurrent connections.
