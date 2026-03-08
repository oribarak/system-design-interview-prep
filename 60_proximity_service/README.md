# System Design Interview: Proximity Service (Nearby Friends) -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We are designing a proximity service that lets users see which of their friends are nearby in real time. The system must continuously ingest location updates from millions of users, compute pairwise proximity between friends, and push notifications when friends come within a configurable distance. I will focus on the location ingestion pipeline, the spatial indexing strategy for efficient proximity detection, and the real-time notification delivery via persistent connections."

## 1) Clarifying Questions (2-5 min)

1. **Definition of nearby**: What distance threshold? -- Configurable per user, default 5 km.
2. **Real-time or periodic**: How fresh should the data be? -- Near real-time, updates within 10-30 seconds.
3. **Opt-in**: Can users control visibility? -- Yes, users opt in to share location and choose who can see them.
4. **Scale**: How many users? -- 500 million total, 100 million with feature enabled, 10 million concurrent.
5. **Friend count**: Average friends per user? -- 500 friends on average.
6. **Location update frequency**: How often does the app send location? -- Every 30 seconds when the app is in the foreground.
7. **Notification**: Push notification or in-app display? -- Both: in-app list of nearby friends + push notification when a friend enters proximity.
8. **Battery/bandwidth**: Concerns about mobile resource usage? -- Yes, must minimize battery impact.

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Users with feature enabled | 100M |
| Concurrent active users | 10M |
| Location updates per user | 1 per 30 seconds |
| Total location updates/sec | 10M / 30 = 333K updates/sec |
| Average friends per user | 500 |
| Average online friends | ~10% of 500 = 50 |
| Proximity checks per update | 50 (check against each online friend) |
| Proximity checks/sec | 333K x 50 = 16.7M checks/sec |
| Location record size | user_id (8B) + lat (8B) + lng (8B) + timestamp (8B) = 32 bytes |
| Active location store size | 10M users x 32 bytes = 320 MB (fits in memory) |
| WebSocket connections | 10M concurrent |

## 3) Requirements (3-5 min)

### Functional Requirements
- **Location Sharing**: Users opt in to share their real-time location with friends.
- **Nearby Friends List**: Display a list of friends currently within the user's proximity threshold, with distance and last-seen time.
- **Proximity Alerts**: Push notification when a friend enters the user's proximity radius.
- **Privacy Controls**: Users can turn location sharing on/off, set visibility per friend or friend list, configure proximity threshold.
- **Location History**: Optionally show a friend's last known location (with permission).

### Non-Functional Requirements
- **Latency**: Nearby friends list should update within 30 seconds of a friend's location change.
- **Throughput**: Handle 333K location updates/sec and 16.7M proximity checks/sec.
- **Scalability**: Support 10M concurrent users.
- **Battery Efficiency**: Minimize battery drain from location updates and network usage.
- **Privacy**: Location data encrypted, access-controlled, and ephemeral (not permanently stored).

## 4) API Design (2-4 min)

### Update Location
```
POST /location
Body: { user_id, lat, lng, accuracy_m, timestamp }
Response: { status: "ok" }
```
(In practice, sent over an existing WebSocket connection rather than individual HTTP requests.)

### Get Nearby Friends
```
GET /nearby-friends?user_id=<id>
Response: {
  friends: [
    { user_id, name, lat, lng, distance_km, last_updated },
    ...
  ]
}
```

### Update Privacy Settings
```
PUT /settings/location
Body: { sharing_enabled: bool, proximity_radius_km: float, visible_to: "all_friends" | "list" | "none", friend_list_id: string }
```

### WebSocket Channel
```
ws://proximity.example.com/ws?user_id=<id>&token=<auth>
Server pushes: { type: "friend_nearby", friend_id, distance_km, lat, lng }
Server pushes: { type: "friend_left", friend_id }
Client sends: { type: "location_update", lat, lng, timestamp }
```

## 5) Data Model (3-5 min)

### User Location (Ephemeral)
| Field | Type | Notes |
|---|---|---|
| user_id | uint64 | Key |
| lat | double | Current latitude |
| lng | double | Current longitude |
| timestamp | timestamp | Last update time |
| sharing_enabled | bool | Is user sharing location |

Stored in: Redis or in-memory cache. TTL of 5 minutes (stale = user is offline).

### Friendship Graph
| Field | Type | Notes |
|---|---|---|
| user_id | uint64 | |
| friend_id | uint64 | |
| bidirectional | bool | Confirmed friendship |

Stored in: Distributed key-value store or graph database. Cached heavily.

### Privacy Settings
| Field | Type | Notes |
|---|---|---|
| user_id | uint64 | |
| sharing_enabled | bool | |
| proximity_radius_km | float | Default 5 km |
| visibility | enum | ALL_FRIENDS, CUSTOM_LIST, NONE |
| blocked_users | list<uint64> | Users who cannot see this user |

Stored in: PostgreSQL, cached in Redis.

### Storage Choices
- **Active locations**: Redis or custom in-memory store. Entire dataset (320 MB) fits in memory.
- **Friendship graph**: Sharded by user_id in a key-value store. Cached in application-level cache.
- **Settings**: PostgreSQL with Redis cache (infrequently changed).

## 6) High-Level Architecture (5-8 min)

**One-sentence summary**: Mobile clients send location updates over persistent WebSocket connections to a location ingestion service, which stores positions in a distributed in-memory cache and publishes updates to a proximity computation layer that checks each user's online friends for proximity, pushing notifications back through the WebSocket channel.

### Components

1. **WebSocket Gateway** -- Manages persistent connections with mobile clients. Routes location updates inbound and proximity notifications outbound.
2. **Location Ingestion Service** -- Receives location updates, writes to the Location Cache, and publishes events.
3. **Location Cache** -- In-memory distributed store (Redis) holding current positions of all active users.
4. **Proximity Computation Service** -- Subscribes to location update events. For each update, checks the user's online friends' positions and computes distances.
5. **Friendship Service** -- Provides the friend list for a given user. Backed by a sharded store with caching.
6. **Notification Dispatcher** -- Sends proximity alerts (friend entered/left radius) via WebSocket and push notifications.
7. **Privacy Service** -- Enforces visibility rules before sending proximity information.
8. **Pub/Sub Channel** -- Kafka or Redis Pub/Sub for distributing location events to proximity workers.

## 7) Deep Dive #1: Proximity Computation at Scale (8-12 min)

### The Naive Approach and Why It Fails

Naive: For every location update, fetch all friends, look up each friend's location, compute distance. With 333K updates/sec and 50 online friends each, that is 16.7M point-to-point distance calculations per second. Each requires a friend list lookup + location cache read.

This is feasible if carefully engineered, but the fan-out (50 reads per update) on the location cache is the bottleneck. Let us optimize.

### Approach: Channel-Based Pub/Sub

Instead of pull-based checking, use a push-based model:

1. **When User A goes online**: Subscribe to location channels of all of User A's online friends. Also, notify all of A's online friends to subscribe to A's channel.
2. **When User A sends a location update**: Publish to User A's channel.
3. **All subscribers (A's friends) receive the update**: Each subscriber computes the distance from their own location to A's new location. If within proximity threshold, display or notify.
4. **When User A goes offline**: Unsubscribe from all friend channels. Notify friends to unsubscribe from A's channel.

### Why This Works Better

- Location updates are published once and delivered to interested subscribers via Pub/Sub. No redundant reads.
- Distance computation happens on the subscriber's server (or even the client device), distributing the compute.
- Memory-efficient: each user's channel has ~50 subscribers (online friends). Total channels: 10M. Total subscriptions: 10M x 50 = 500M.

### Pub/Sub Implementation

Use a distributed Pub/Sub system. Options:
- **Redis Pub/Sub**: Simple, but single-server channels do not shard well.
- **Custom channel routing**: Hash user_id to a Pub/Sub server. Each server manages channels for a subset of users. When A subscribes to B's channel and B is on a different server, create a cross-server subscription.
- **Alternative: Kafka with per-user topics**: Too many topics (10M). Not practical.

**Chosen approach**: Consistent hash ring of Pub/Sub servers. User A's channel lives on server H(A). Friends of A connect to H(A) to subscribe. Each server handles ~10M / N channels.

### Geospatial Optimization

Even with Pub/Sub, each subscriber must compute distance. Optimization:
- **Geohash bucketing**: Divide the world into geohash cells (e.g., precision 6 = ~1.2 km cells). Each user is assigned to a geohash cell based on their location.
- **Coarse filter**: If two users are in the same cell or adjacent cells, they could be within 5 km. Only compute exact distance for these pairs.
- **Skip distant friends**: If a friend's geohash cell is far away (> 2 cells apart at precision 6), skip the distance calculation entirely.
- This reduces actual distance computations by 90%+ since most friends are not nearby.

### Handling Connection Churn

- Users go online/offline frequently. Each transition triggers subscribe/unsubscribe operations.
- Batch subscribe operations: when a user comes online, send a single bulk subscribe request for all 50 online friends' channels.
- TTL on subscriptions: if a user stops sending location updates for 5 minutes, automatically unsubscribe and mark offline.

## 8) Deep Dive #2: WebSocket Infrastructure at Scale (5-8 min)

### 10 Million Concurrent WebSocket Connections

Each WebSocket connection is a long-lived TCP connection consuming memory on the server.

- **Memory per connection**: ~20 KB (buffers, state). 10M connections x 20 KB = 200 GB total.
- **Server capacity**: A single server can handle ~100K concurrent WebSocket connections (with 2 GB allocated to connection state).
- **Server count**: 10M / 100K = 100 WebSocket gateway servers.

### Connection Routing

- User connects to the nearest WebSocket gateway (via DNS geo-routing or an L4 load balancer with least-connections).
- A connection registry (Redis) maps user_id -> gateway_server_id. This is needed so the proximity service knows which gateway to push notifications through.
- When the proximity service determines that User B should be notified about User A's proximity, it looks up B's gateway in the registry and sends the notification to that gateway.

### Graceful Handling of Server Failures

- If a gateway server dies, all its connections drop. Clients reconnect automatically (with exponential backoff).
- The connection registry has a TTL; stale entries expire.
- No data loss since location updates are ephemeral and clients will re-send.

### Mobile Optimization

- **Background mode**: When the app is backgrounded, switch from WebSocket to periodic HTTP location updates (every 5 minutes) using OS-level background fetch.
- **Significant location change**: Use OS APIs (iOS Significant Location Change, Android Fused Location) to trigger updates only when the user moves significantly (> 500 m).
- **Batch updates**: On reconnection, send the latest location in the reconnect handshake rather than waiting for the next update interval.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Proximity computation | Pub/Sub (push-based) | Pull-based (check all friends per update) | Push reduces redundant reads; compute is distributed |
| Pub/Sub infrastructure | Custom channel servers (consistent hashing) | Redis Pub/Sub | Redis Pub/Sub does not shard well for 10M channels |
| Coarse filtering | Geohash bucketing | Quadtree | Geohash is simpler, integrates with Redis, sufficient for proximity checks |
| Client connection | WebSocket | SSE / Long polling | WebSocket supports bidirectional (location up, notifications down) |
| Location storage | In-memory (Redis) | Database | Ephemeral data; latency-sensitive; 320 MB fits easily in memory |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100M concurrent users)
- 1,000 WebSocket gateway servers. Geo-distributed across regions.
- Pub/Sub server ring scales to hundreds of nodes.
- Shard friendship cache by user_id. Pre-compute online friend lists.
- Geohash-based filtering becomes critical at this scale to reduce computation.

### 100x (1B concurrent users)
- Regional deployment: users only compute proximity with friends in the same region/country.
- Hierarchical Pub/Sub: regional brokers aggregate cross-region subscriptions.
- Move distance computation to client devices entirely: server pushes friend locations, client computes distance locally.
- Approximate proximity: use geohash cell membership as a coarse proximity signal, exact distance only on demand.

## 11) Reliability and Fault Tolerance (3-5 min)

- **WebSocket gateway failure**: Clients reconnect to another gateway. Connection registry updated. No permanent state on gateways.
- **Pub/Sub server failure**: Channel ownership moves to next node on consistent hash ring. Subscribers reconnect. Brief gap in updates (seconds).
- **Location cache failure**: Redis cluster with replicas. Failover to replica. Worst case: users' locations are stale for a few seconds.
- **Graceful degradation**: If proximity computation is overloaded, increase update interval (30s -> 60s). Show stale nearby-friends list rather than nothing.
- **Privacy safety net**: If privacy service is down, default to not showing location (fail-closed, not fail-open).

## 12) Observability and Operations (2-4 min)

- **Connection metrics**: Active WebSocket connections, connection rate, disconnection rate, connections per server.
- **Location update throughput**: Updates/sec ingested, Pub/Sub messages delivered/sec.
- **Proximity computation**: Checks/sec, geohash filter hit rate (% of pairs skipped), latency per update.
- **User experience**: Time from location update to friend seeing the update (end-to-end latency).
- **Battery impact**: Track client-side battery consumption attributed to location updates via analytics SDK.
- **Alerting**: Page on WebSocket server connection count > 120K (capacity limit), Pub/Sub delivery lag > 30 seconds.

## 13) Security (1-3 min)

- **Location privacy**: Location data is ephemeral (TTL 5 min). Not stored permanently. User must explicitly opt in.
- **Access control**: Location visible only to confirmed friends with mutual consent. Privacy service enforces per-friend visibility.
- **Encryption**: TLS for all WebSocket connections. Location data encrypted in transit.
- **Stalking prevention**: Rate-limit location queries per user. Anomaly detection for users requesting location data at unusual rates.
- **GDPR compliance**: Users can delete all location history. Location data is not used for advertising or shared with third parties.

## 14) Team and Operational Considerations (1-2 min)

- **Real-time infrastructure team**: Owns WebSocket gateways, Pub/Sub servers, connection registry.
- **Location team**: Owns location ingestion, caching, geospatial indexing.
- **Product team**: Owns proximity logic, privacy settings, user experience.
- **Mobile team**: Owns client-side location SDK, battery optimization, background behavior.
- **On-call**: Shared rotation between real-time infra and location teams.

## 15) Common Follow-up Questions

1. **How do you handle friends in different time zones?** -- Not relevant to proximity (physical distance), but display "last active" times in the friend's local time zone for context.
2. **Can you show friends on a map?** -- Yes, the client already receives lat/lng of nearby friends. Display as pins on a map view with distance labels.
3. **How do you handle location spoofing?** -- Detect anomalies: impossible travel speed, mismatched IP geolocation vs GPS, jailbroken devices. Flag and suppress spoofed locations.
4. **What about privacy laws in different countries?** -- Region-specific consent flows. In the EU, explicit opt-in required. Store location data in-region. Honor right-to-deletion requests.
5. **How do you handle users with thousands of friends?** -- Cap the proximity check to closest 500 friends. Use geohash pre-filtering aggressively. Shard channel subscriptions.

## 16) Closing Summary (30-60s)

"We designed a proximity service that handles 10 million concurrent users by using a push-based Pub/Sub model where each user's location updates are published to a channel subscribed by their online friends. Geohash bucketing eliminates 90% of distance computations, and 100 WebSocket gateway servers maintain persistent connections for sub-30-second notification delivery. The system uses ephemeral in-memory storage for locations, fail-closed privacy enforcement, and degrades gracefully by increasing update intervals under load."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Guidance |
|---|---|---|
| Location update interval | 30 seconds | Lower = fresher but more battery/bandwidth |
| Proximity radius default | 5 km | User-configurable |
| Geohash precision | 6 (~1.2 km cells) | Lower = coarser filter, fewer checks; higher = more precise |
| Location TTL | 5 minutes | After this, user is considered offline |
| WebSocket ping interval | 30 seconds | Keep-alive for connection health |
| Max concurrent connections per server | 100K | Based on server memory (2 GB for connections) |
| Background update interval | 5 minutes | OS-level background fetch limit |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| Geohash | Encoding of geographic coordinates into a short string for spatial indexing |
| Pub/Sub | Publish/Subscribe messaging pattern for event distribution |
| Consistent Hashing | Distributed hash technique that minimizes key redistribution on node changes |
| Significant Location Change | OS API that triggers callbacks only on meaningful position changes |
| Connection Registry | Mapping of user_id to the WebSocket server handling their connection |
| Ephemeral Data | Temporary data with a TTL that is not permanently stored |
| Fail-Closed | Default behavior that denies access when a safety system fails |

## Appendix C: References

- Facebook Engineering: "Nearby Friends" system design (2014)
- Redis documentation: Geospatial commands (GEOADD, GEOSEARCH)
- Nishtala et al. "Scaling Memcache at Facebook" (NSDI 2013)
- WebSocket RFC 6455
- Geohash encoding: https://en.wikipedia.org/wiki/Geohash
