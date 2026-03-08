# System Design Interview: WebSocket Notification System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a real-time notification system that pushes events to users the instant they happen -- think likes, comments, mentions, order updates, or breaking alerts. The core challenge is maintaining millions of persistent WebSocket connections while delivering notifications with sub-second latency, exactly once, and in order. I will walk through the connection management layer, the fan-out pipeline, presence tracking, and how we guarantee delivery even when users are offline. Let me start by clarifying the scope."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing a general-purpose notification platform (multi-tenant) or an internal service for a single product? *Assume multi-tenant platform that multiple product teams plug into.*
- Do we need to support both in-app (WebSocket) and out-of-band channels (push, email, SMS)? *Focus on WebSocket; mention push as extension.*
- Should notifications be persistent (viewable later) or ephemeral (fire-and-forget)? *Persistent with read/unread state.*

**Scale**
- How many concurrent connected users at peak? *Target 50 million concurrent WebSocket connections.*
- What is the notification throughput? *500K notifications/sec at peak across all users.*
- Average notifications per user per day? *~50 notifications/day.*

**Policy / Constraints**
- Do we need guaranteed delivery (at-least-once) or best-effort? *At-least-once with client-side dedup.*
- What is the acceptable end-to-end latency from event to user screen? *p99 < 500ms.*
- Are there rate-limiting or batching requirements to avoid overwhelming users? *Yes, aggregate similar notifications (e.g., "3 people liked your post").*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Total users | Given | 500M |
| DAU | 20% of total | 100M |
| Peak concurrent connections | 50% of DAU | 50M |
| Notifications/day | 100M x 50 | 5B/day |
| Notifications/sec (avg) | 5B / 86400 | ~58K/s |
| Peak notifications/sec | 10x average | ~500K/s |
| Notification payload size | avg 500 bytes | 500B |
| Peak bandwidth (outbound) | 500K x 500B | 250 MB/s |
| Storage per notification | 500B + metadata ~1KB | 1KB |
| Storage/day | 5B x 1KB | 5 TB/day |
| Storage/year (with 90-day retention) | 5TB x 90 | 450 TB |
| WebSocket servers needed | 50M / 100K per server | 500 servers |

**Memory for connection state:** 50M connections x 2KB state = 100 GB total across 500 servers = 200 MB/server -- very manageable.

## 3) Requirements (3-5 min)

### Functional
- **Real-time delivery**: Push notifications to connected users within sub-second latency via WebSocket.
- **Offline buffering**: Store undelivered notifications and replay them when the user reconnects.
- **Read/unread state**: Track per-user read status; support "mark all as read."
- **Notification aggregation**: Collapse similar notifications (e.g., "Alice and 4 others liked your photo").
- **Multi-device sync**: If a user is connected on phone and laptop, both receive the notification; marking read on one marks read everywhere.
- **Subscription management**: Users can mute or configure notification preferences per category.

### Non-functional
- **Low latency**: p99 end-to-end < 500ms.
- **High availability**: 99.99% uptime -- notifications are a core engagement driver.
- **Scalability**: Handle 50M concurrent connections and 500K notifications/sec.
- **Exactly-once semantics (user-perceived)**: At-least-once delivery with client-side idempotency.
- **Ordering**: Notifications for a single user appear in causal order.
- **Fault tolerance**: No single point of failure; graceful degradation under load.

## 4) API Design (2-4 min)

### WebSocket Protocol

```
# Client -> Server
CONNECT /ws/notifications?token=<jwt>
SUBSCRIBE {"channels": ["user:<uid>", "room:<rid>"]}
ACK {"notification_id": "n123"}
MARK_READ {"notification_ids": ["n123", "n124"]}

# Server -> Client
NOTIFICATION {"id":"n123","type":"like","body":{...},"ts":1700000000}
BATCH_NOTIFICATION {"notifications": [...], "has_more": true}
PRESENCE {"user_id":"u456","status":"online"}
```

### REST API (for producers and history)

```
POST /api/v1/notifications/send
  Body: {user_ids: [], type, title, body, data, priority, collapse_key}
  Returns: {notification_id, status}

GET /api/v1/notifications?user_id=<uid>&cursor=<cursor>&limit=20
  Returns: {notifications: [...], next_cursor}

PUT /api/v1/notifications/read
  Body: {notification_ids: [] | "all", before_ts}
  Returns: {updated_count}

GET /api/v1/notifications/unread-count?user_id=<uid>
  Returns: {count}

PUT /api/v1/notifications/preferences
  Body: {user_id, channel_preferences: {likes: true, comments: true, ...}}
  Returns: {ok}
```

## 5) Data Model (3-5 min)

### Notifications Table (Cassandra / ScyllaDB)
| Column | Type | Notes |
|--------|------|-------|
| user_id | UUID | Partition key |
| notification_id | TIMEUUID | Clustering key (DESC) -- natural time ordering |
| type | VARCHAR | like, comment, mention, system |
| sender_id | UUID | Who triggered the notification |
| title | TEXT | Display title |
| body | TEXT | Display body |
| data | JSON | Arbitrary payload (deep link, image URL, etc.) |
| collapse_key | VARCHAR | For aggregation (e.g., "like:post:123") |
| is_read | BOOLEAN | Read status |
| created_at | TIMESTAMP | Event time |

**Why Cassandra?** Write-heavy workload (5B/day), partition by user_id gives single-partition reads for a user's notification feed, and TTL handles auto-expiry at 90 days.

### Connection Registry (Redis Cluster)
| Key Pattern | Value | TTL |
|-------------|-------|-----|
| `conn:{user_id}` | SET of `{server_id}:{conn_id}` | Heartbeat-refreshed, 60s |
| `server:{server_id}:connections` | Count of active connections | 60s |
| `user:{user_id}:unread_count` | Integer counter | None |

### User Preferences (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| user_id | UUID | PK |
| preferences | JSONB | Per-category mute/unmute settings |
| quiet_hours_start | TIME | Do-not-disturb window |
| quiet_hours_end | TIME | |
| updated_at | TIMESTAMP | |

## 6) High-Level Architecture (5-8 min)

**Dataflow:** Producer services emit notification events to a message broker. The Notification Router consumes events, checks user preferences, performs aggregation, writes to the notification store, then looks up the user's active connections in the Connection Registry. It publishes the notification to the correct WebSocket Gateway server(s) via an internal pub/sub channel. The Gateway pushes the message down the WebSocket to the client. The client ACKs, and the Router marks the notification as delivered.

**Components:**
- **WebSocket Gateway**: Stateful edge servers holding persistent connections. Handles connect/disconnect, heartbeat, and message framing.
- **Connection Registry (Redis)**: Maps user_id to the set of gateway servers where the user is connected.
- **Notification Router**: Stateless workers that consume events, apply preferences/aggregation, and route to the correct gateway.
- **Message Broker (Kafka)**: Durable event bus between producers and routers; partitioned by user_id for ordering.
- **Aggregation Service**: Collapses notifications with the same collapse_key within a time window.
- **Notification Store (Cassandra)**: Persistent storage for notification history and read state.
- **Preference Service**: Caches user notification preferences; filters out muted categories.
- **Unread Counter (Redis)**: Atomic counter per user for badge counts.
- **Load Balancer (L4)**: Distributes WebSocket connections across gateway servers using consistent hashing on user_id.

**One-sentence summary:** Producers publish events to Kafka, stateless routers fan out to the correct WebSocket gateway servers via Redis pub/sub after applying preferences and aggregation, and gateways push notifications down persistent connections to clients.

## 7) Deep Dive #1: Connection Management and Fan-out at Scale (8-12 min)

### The Challenge
Maintaining 50M concurrent WebSocket connections across 500 servers and routing a notification to exactly the right server(s) in sub-100ms is the hardest part of this system.

### Connection Lifecycle

1. **Connect**: Client opens a WebSocket to the Gateway via L4 load balancer. The Gateway authenticates the JWT, registers `conn:{user_id} -> {server_id}:{conn_id}` in Redis with a 60s TTL, and replays any buffered notifications since the client's last seen timestamp.
2. **Heartbeat**: Client sends a ping every 30s. Gateway refreshes the Redis TTL. If no ping for 60s, the connection is considered dead and cleaned up.
3. **Disconnect**: Gateway removes the entry from Redis and publishes a presence event.

### Fan-out Strategy

We use a **two-tier fan-out**:

**Tier 1 -- Kafka to Router (user-level fan-out):**
- Notification events arrive on Kafka partitioned by `user_id`.
- For 1-to-1 notifications (e.g., "Alice liked your photo"), the event targets a single user -- simple lookup.
- For broadcast notifications (e.g., "breaking news"), a Fan-out Service pre-computes the recipient list and publishes individual per-user events to Kafka. This is the "fan-out on write" approach.

**Tier 2 -- Router to Gateway (server-level routing):**
- The Router looks up `conn:{user_id}` in Redis to find which gateway server(s) hold the user's connections.
- If the user has connections on servers A and B (multi-device), the Router publishes the notification to both via Redis pub/sub channels `gateway:{server_id}`.
- Each Gateway subscribes to its own channel and pushes to the local WebSocket connection.

### Why not direct Kafka-to-Gateway?
Each gateway would need to consume all partitions to check if it holds the target user's connection. With 500 servers, that is 500x read amplification. The Redis lookup makes routing O(1).

### Handling Hot Users
Some users (celebrities) might receive thousands of notifications per second. We handle this with:
- **Aggregation**: Collapse "X liked your post" into "X and 999 others liked your post" using the collapse_key with a 5-second tumbling window.
- **Rate limiting**: Cap at 10 notifications/sec per user; excess is batched.
- **Priority queuing**: High-priority notifications (DMs, mentions) bypass rate limits.

### Guaranteed Delivery
- Every notification gets a unique `notification_id` (TIMEUUID).
- The Gateway sends the notification and starts a 5-second ACK timer.
- If the client ACKs, the notification is marked delivered.
- If no ACK, the Gateway retries up to 3 times with exponential backoff.
- If the connection drops before ACK, the notification remains in the "undelivered" state in Cassandra.
- On reconnect, the client sends its `last_seen_notification_id`, and the server replays all newer notifications.
- Client-side dedup uses the `notification_id` to discard duplicates.

### Connection Draining for Deploys
During rolling deploys, a gateway server:
1. Stops accepting new connections (removed from LB).
2. Sends a `RECONNECT` control frame to all connected clients with a random jitter (0-30s) to avoid thundering herd.
3. Waits for all connections to drain (max 60s).
4. Shuts down.

## 8) Deep Dive #2: Notification Aggregation and Ordering (5-8 min)

### Aggregation

Many notification systems drown users in noise. Aggregation is critical for UX.

**Mechanism:**
- Each notification carries a `collapse_key` (e.g., `like:post:456`).
- The Aggregation Service maintains a per-user, per-collapse-key buffer in Redis: `agg:{user_id}:{collapse_key}`.
- When a new notification arrives:
  1. Increment the counter for that collapse_key.
  2. If count == 1, start a 5-second timer.
  3. If count > threshold (e.g., 3), immediately flush an aggregated notification.
  4. When the timer fires, flush whatever has accumulated.
- The aggregated notification is: "Alice, Bob, and 2 others liked your post."

**Trade-off:** Aggregation adds 0-5s latency. For high-priority types (DMs), we skip aggregation entirely.

### Ordering

Users expect notifications in chronological order. Challenges:
- Multiple producers may emit events for the same user concurrently.
- Network delays can cause out-of-order delivery.

**Solution:**
- Kafka partitioning by `user_id` ensures all events for a user are processed by the same Router instance in order.
- Each notification carries a monotonically increasing `sequence_number` per user (generated by an atomic Redis counter: `seq:{user_id}`).
- The client maintains a local sequence counter. If it receives seq=5 but hasn't seen seq=4, it buffers seq=5 and requests a gap fill from the REST API.
- For multi-device, each device independently tracks its sequence position.

## 9) Trade-offs and Alternatives (3-5 min)

### WebSocket vs. SSE vs. Long Polling
| Approach | Pros | Cons |
|----------|------|------|
| WebSocket | Bidirectional, low overhead per message | Stateful, harder to scale |
| SSE | Simpler, HTTP-based, auto-reconnect | Unidirectional, limited browser connections |
| Long Polling | Works everywhere, stateless servers | Higher latency, more overhead |

We chose WebSocket because we need bidirectional communication (ACKs, subscriptions) and the lowest possible per-message overhead at 500K msg/s.

### Fan-out on Write vs. Fan-out on Read
- **Fan-out on write** (our approach): Pre-compute recipients and push. Good for real-time delivery but expensive for broadcast to millions.
- **Fan-out on read**: Store the event once, and each user pulls on connect. Lower write cost but higher read latency.
- We use fan-out on write for targeted notifications and fan-out on read for low-priority broadcasts (e.g., system announcements).

### Redis Pub/Sub vs. Dedicated Message Bus for Gateway Routing
- Redis pub/sub is simpler and lower latency but messages are fire-and-forget (no persistence).
- Since we already have guaranteed delivery via Cassandra + reconnect replay, Redis pub/sub is acceptable for the hot path.

### CAP / PACELC
- **Connection Registry (Redis)**: AP -- eventual consistency is fine; a stale entry just causes a failed push, which triggers retry via the durable path.
- **Notification Store (Cassandra)**: AP with tunable consistency; we use LOCAL_QUORUM for writes, ONE for reads.
- **PACELC**: Under partition, we choose availability (deliver notifications even if some metadata is stale). Under normal operation, we choose low latency over strict consistency.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (500M concurrent connections, 5M notifications/sec)
- **WebSocket Gateways**: Scale from 500 to 5,000 servers. Use geographic regions to keep connections local.
- **Kafka**: Increase partitions proportionally. Separate topics for high-priority vs. bulk notifications.
- **Connection Registry**: Shard Redis across more nodes. Consider switching to a dedicated connection registry service with in-memory sharding.
- **Aggregation**: More aggressive batching to reduce per-user fan-out.

### 100x (5B concurrent connections, 50M notifications/sec)
- **Edge PoPs**: Deploy WebSocket gateways at edge locations worldwide. Each PoP handles local connections and communicates with the central notification router via a backbone.
- **Hierarchical fan-out**: Instead of flat Redis pub/sub, use a tree structure -- central router fans out to regional routers, which fan out to local gateways.
- **Custom protocol**: Replace WebSocket with a custom binary protocol (like QUIC-based) for lower overhead and better multiplexing.
- **Tiered storage**: Hot notifications in Redis (last 24h), warm in Cassandra (90 days), cold in object storage.

### Cost at Scale
At 500 servers for 50M connections: ~$500K/month in compute alone. At 100x, geographic distribution and edge caching become essential to keep costs manageable.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Gateway server crash | ~100K users lose connections | Clients auto-reconnect to another server; unACKed notifications replayed from Cassandra |
| Redis cluster node failure | Partial connection registry loss | Redis Cluster auto-failover; clients reconnect and re-register |
| Kafka broker failure | Temporary notification delay | Kafka replication factor 3; consumers failover to new leader |
| Cassandra node failure | Degraded write/read throughput | RF=3, LOCAL_QUORUM writes; hinted handoff for recovery |
| Network partition between regions | Cross-region notifications delayed | Each region operates independently for local users; cross-region notifications queue until partition heals |

### Graceful Degradation
1. **Under extreme load**: Shed low-priority notifications (marketing, recommendations) first. Always deliver DMs and security alerts.
2. **If WebSocket layer is overwhelmed**: Fall back to polling mode -- client switches to REST polling at 10s intervals.
3. **If aggregation service fails**: Deliver notifications individually (worse UX but no data loss).

### Disaster Recovery
- Kafka topics replicated across AZs with `min.insync.replicas=2`.
- Cassandra uses multi-DC replication.
- Redis connection registry is reconstructed from live heartbeats within 60s after total loss.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Connection count** per gateway server (alert if > 120K per server).
- **Notification delivery latency** (p50, p95, p99) -- alert if p99 > 500ms.
- **Delivery success rate** -- alert if ACK rate drops below 99%.
- **Unread count accuracy** -- periodic reconciliation between Redis counter and Cassandra ground truth.
- **Kafka consumer lag** per partition -- alert if lag > 10K messages.
- **WebSocket error rate** (connection failures, frame errors).

### Dashboards
- Real-time connection heatmap by region and server.
- Notification throughput (produced vs. delivered vs. ACKed) over time.
- Fan-out latency breakdown: Kafka -> Router -> Redis Pub/Sub -> Gateway -> Client ACK.

### Runbooks
- **High consumer lag**: Scale up Router instances; check for hot partitions.
- **Gateway memory pressure**: Trigger connection drain on overloaded servers.
- **Spike in reconnections**: Check for network issues or a bad deploy causing connection drops.

## 13) Security (1-3 min)

- **Authentication**: JWT-based auth on WebSocket connect. Token includes user_id, scopes, and expiry. Tokens refreshed via a side-channel REST call before expiry.
- **Authorization**: Users can only subscribe to their own notification channels. Room/group channels require membership verification.
- **Transport encryption**: WSS (WebSocket over TLS) enforced. No plaintext WebSocket connections.
- **Rate limiting**: Per-user connection limit (max 5 concurrent devices). Per-producer rate limit on the send API (1000 notifications/sec per API key).
- **Abuse prevention**: Notification content sanitized to prevent XSS in web clients. Producers authenticated with API keys and scoped to specific notification types.
- **DDoS protection**: L4 load balancer with SYN flood protection. Connection rate limiting per IP (max 10 new connections/sec per IP).

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Platform team owns the Gateway and Router (3-4 engineers). A separate team owns the Notification Store and Preference Service. Product teams are consumers that integrate via the send API.
- **Deployment**: Rolling deploys with connection draining. Canary 5% of gateway servers for 15 minutes before full rollout.
- **On-call**: Two-tier on-call. Tier 1 monitors dashboards and handles connection drain / scaling. Tier 2 (senior) handles data consistency issues and Kafka recovery.
- **Rollout plan**: Start with a single product (e.g., social feed likes). Measure delivery latency and ACK rates. Expand to all notification types over 4 weeks.

## 15) Common Follow-up Questions

**Q: How do you handle notification preferences that change while notifications are in-flight?**
A: The Preference Service is checked at routing time, not at event generation. If a user mutes "likes" while a like notification is in the Kafka queue, the Router will filter it out when it processes the event. There is a small race window (~seconds), which is acceptable.

**Q: How do you handle users connected to multiple regions?**
A: The Connection Registry is globally replicated (Redis with cross-region sync). The Router queries the registry and publishes to gateways in all regions where the user has active connections. We accept ~50ms additional latency for cross-region delivery.

**Q: What happens during a thundering herd (e.g., a celebrity posts and millions get notified)?**
A: The Fan-out Service pre-computes recipients in batches of 10K, pacing the writes to Kafka to avoid overwhelming downstream. High-fanout events are rate-limited to spread delivery over 30-60 seconds. Users closest to the celebrity (frequent engagers) get notifications first.

**Q: Can you support end-to-end encryption for notifications?**
A: Yes, but it shifts complexity to the client. The producer encrypts the notification payload with the recipient's public key. The notification system treats the payload as opaque bytes. Aggregation becomes impossible on encrypted payloads, so the client must handle it locally.

**Q: How do you test this at scale before launch?**
A: Shadow traffic from production, replayed through the new system without actually pushing to clients. Load testing with synthetic WebSocket connections (tools like Artillery or custom Go clients) simulating 1M concurrent connections per test server.

## 16) Closing Summary (30-60s)

> "We designed a real-time WebSocket notification system that maintains 50M concurrent connections across 500 gateway servers, delivers 500K notifications per second with p99 latency under 500ms, and guarantees at-least-once delivery with client-side dedup for exactly-once user experience. The architecture uses Kafka for durable event ingestion partitioned by user_id, stateless Router workers for preference filtering and aggregation, Redis for connection registry and pub/sub routing to the correct gateway, and Cassandra for persistent notification storage with 90-day TTL. Key design decisions include two-tier fan-out (Kafka for user-level, Redis pub/sub for server-level), a 5-second aggregation window to reduce notification noise, and connection draining with jittered reconnect for zero-downtime deploys. The system degrades gracefully under load by shedding low-priority notifications and falling back to polling mode."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `max_connections_per_server` | 100,000 | 50K-200K | Memory vs. server count trade-off |
| `heartbeat_interval_sec` | 30 | 10-60 | Stale connection detection speed vs. bandwidth |
| `heartbeat_ttl_sec` | 60 | 30-120 | Must be > heartbeat_interval |
| `aggregation_window_sec` | 5 | 1-30 | Batching latency vs. notification noise |
| `aggregation_threshold` | 3 | 2-10 | Min notifications before immediate flush |
| `ack_timeout_sec` | 5 | 2-15 | Retry sensitivity vs. duplicate risk |
| `max_retries` | 3 | 1-5 | Delivery guarantee vs. load |
| `notification_ttl_days` | 90 | 30-365 | Storage cost vs. history depth |
| `max_devices_per_user` | 5 | 1-10 | Fan-out cost per user |
| `kafka_consumer_threads` | 16 | 4-64 | Router throughput vs. CPU |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **WebSocket** | Full-duplex communication protocol over a single TCP connection |
| **Fan-out** | Pattern of distributing a single message to multiple recipients |
| **Collapse key** | Identifier used to group related notifications for aggregation |
| **Connection draining** | Gracefully migrating connections off a server before shutdown |
| **Thundering herd** | Surge of reconnections when many clients try to reconnect simultaneously |
| **At-least-once delivery** | Guarantee that a message is delivered one or more times (may duplicate) |
| **Idempotency** | Property where repeating an operation has no additional effect |
| **Consumer lag** | How far behind a Kafka consumer is from the latest produced message |
| **Sequence number** | Monotonically increasing counter for ordering guarantees |
| **Presence** | Knowledge of whether a user is currently online or offline |

## Appendix C: References

- RFC 6455 -- The WebSocket Protocol
- Kafka documentation on consumer groups and partitioning
- Discord Engineering Blog: "How Discord Handles Push Request Routing"
- Facebook Engineering: "Building Mobile-First Infrastructure for Messenger"
- LinkedIn Engineering: "Real-time Presence Platform"
- Cassandra data modeling best practices for time-series-like workloads
