# WebSocket Notification System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to push real-time notifications to millions of connected users the instant events happen -- likes, comments, order updates. The hard part is maintaining 50M persistent WebSocket connections while routing each notification to the exact server holding the target user's connection in sub-second latency.

## 2. Requirements
**Functional:** Push notifications to connected users in real time, buffer notifications for offline users and replay on reconnect, track read/unread state, and aggregate similar notifications (e.g., "Alice and 4 others liked your post").

**Non-functional:** p99 end-to-end latency under 500ms, 99.99% uptime, 50M concurrent connections, at-least-once delivery with client-side dedup for exactly-once perception.

## 3. Core Concept: Two-Tier Fan-Out
The key insight is separating user-level fan-out from server-level routing. Kafka handles the first tier: notification events partitioned by user_id ensure per-user ordering. Redis handles the second tier: a connection registry maps each user_id to the gateway server(s) holding their WebSocket, so the router publishes directly to the right server's channel instead of broadcasting to all 500 servers.

## 4. High-Level Architecture
```
Producers --> [Kafka] --> Router Workers --> [Redis Pub/Sub] --> WS Gateways --> Clients
                              |                                      |
                        [Preference Svc]                    [Connection Registry]
                              |                                  (Redis)
                        [Notification Store]
                           (Cassandra)
```
- **Kafka**: durable event bus partitioned by user_id for ordering.
- **Router Workers**: stateless; check preferences, aggregate, look up connections, route.
- **WebSocket Gateways**: stateful edge servers holding ~100K connections each.
- **Connection Registry (Redis)**: maps user_id to gateway server(s).
- **Notification Store (Cassandra)**: persistent history for offline replay and read state.

## 5. How Notification Delivery Works
1. Producer publishes a notification event to Kafka (partitioned by target user_id).
2. Router worker consumes the event, checks user preferences, applies aggregation.
3. Router writes the notification to Cassandra and looks up the user's connections in Redis.
4. Router publishes the notification to the correct gateway server(s) via Redis pub/sub.
5. Gateway pushes the notification down the user's WebSocket connection.
6. Client ACKs; if no ACK within 5 seconds, gateway retries up to 3 times.
7. On reconnect, client sends last_seen_notification_id and server replays missed notifications.

## 6. What Happens When Things Fail?
- **Gateway server crashes**: ~100K users auto-reconnect to another server. Un-ACKed notifications are replayed from Cassandra on reconnect.
- **Redis node failure**: Redis Cluster auto-failover. Stale connection entries cause a failed push, but the durable Cassandra path ensures delivery on reconnect.
- **Kafka broker failure**: replication factor 3 with automatic consumer failover to a new partition leader.

## 7. Scaling
- **10x**: Scale gateways from 500 to 5,000 servers across geographic regions. Increase Kafka partitions. More aggressive notification aggregation to reduce per-user fan-out.
- **100x**: Deploy gateways at edge PoPs worldwide. Use hierarchical fan-out (central router to regional routers to local gateways) instead of flat Redis pub/sub. Tiered notification storage (Redis for 24h hot, Cassandra for 90 days).

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| WebSocket over SSE/long-polling | Bidirectional (ACKs, subscriptions) but stateful and harder to scale |
| Fan-out on write (pre-compute recipients) | Low latency delivery but expensive for broadcast to millions |
| Redis pub/sub for gateway routing | Simple and fast but fire-and-forget; durable delivery relies on Cassandra replay |

## 9. Closing (30s)
> We designed a real-time notification system using a two-tier fan-out: Kafka partitions events by user for ordering, and Redis pub/sub routes to the exact gateway server holding each user's WebSocket. Stateless routers apply preferences and aggregation, Cassandra stores notifications for offline replay, and clients ACK for at-least-once delivery with dedup. The system handles 50M connections across 500 gateways and degrades gracefully by shedding low-priority notifications and falling back to polling under extreme load.
