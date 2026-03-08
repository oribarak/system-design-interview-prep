# Proximity Service (Nearby Friends) -- Cheat Sheet

## Key Numbers
- 100M users with feature enabled, 10M concurrent active
- 333K location updates/sec (10M users / 30s interval)
- 50 online friends per user on average
- 320 MB total location data (fits in memory)
- 100 WebSocket gateway servers (100K connections each)
- End-to-end proximity detection latency: 2-5 seconds

## Core Components
- WebSocket Gateway: persistent connections for bidirectional location/notification flow
- Location Cache (Redis): ephemeral in-memory store of all active user positions
- Pub/Sub Channel Servers: consistent-hash-based channel routing; each user has a channel subscribed by online friends
- Proximity Workers: receive friend location updates, apply geohash coarse filter, compute exact distance
- Friendship Service: provides friend lists, cached in Redis
- Privacy Service: enforces per-user visibility rules (fail-closed)

## Architecture in One Sentence
Users publish location updates to personal Pub/Sub channels subscribed by their online friends; proximity workers apply geohash coarse filtering and Haversine distance checks, pushing nearby-friend notifications back through WebSocket gateways within seconds.

## Top 3 Trade-offs
1. Push-based Pub/Sub vs. pull-based polling: push eliminates redundant location reads at the cost of managing millions of channel subscriptions.
2. Geohash coarse filter vs. exact distance for all pairs: geohash skips 90%+ of pairs cheaply, with rare false negatives at cell boundaries handled by checking adjacent cells.
3. WebSocket vs. periodic HTTP polling: WebSocket enables sub-second delivery but requires managing 10M persistent connections.

## Scaling Story
10x: more WebSocket gateways and Pub/Sub nodes, geo-distribute by region, pre-compute online friend lists.
100x: move distance computation to client devices, hierarchical Pub/Sub, approximate proximity via geohash cell membership.

## Closing Statement
A push-based Pub/Sub architecture with geohash-filtered proximity computation, ephemeral in-memory location storage, and 100 WebSocket gateways delivers real-time nearby-friend notifications for 10M concurrent users within 5 seconds of a location change.
