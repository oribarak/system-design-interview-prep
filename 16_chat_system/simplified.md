# Chat System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a general-purpose chat system supporting one-on-one messaging, group chats, and real-time delivery for 100M daily active users. The core challenge is guaranteeing consistent message ordering across all participants in a conversation while handling 50M concurrent WebSocket connections and ensuring zero message loss.

## 2. Requirements
**Functional:** One-on-one and group messaging (up to 500 members) with text and media. Message delivery status (sent, delivered, read). Message history with pagination. Presence and typing indicators. Multi-device sync. Push notifications for offline users.

**Non-functional:** Message delivery in under 200ms for online recipients, zero message loss, consistent message ordering within conversations, 99.99% availability, 50M concurrent connections, 300K peak QPS.

## 3. Core Concept: Per-Conversation Sequence Numbers via Redis INCR
The key insight is how message ordering is solved. Client timestamps cannot be trusted (clock skew), and a global sequence number creates a single bottleneck. Instead, each conversation maintains its own monotonic counter in Redis. When a message arrives, the server atomically increments `conv_seq:{conversation_id}` via Redis INCR, giving the message a definitive sequence number. Since conversations have low individual message rates (under 100/minute at p99), a single Redis key per conversation handles the load easily. Clients detect sequence gaps and self-heal by fetching missing messages via REST. Multi-device sync works by tracking the last received sequence per device.

## 4. High-Level Architecture

```
Client --> [ Chat Server (WebSocket) ] --> [ Redis INCR (sequence) ]
                     |                              |
                     v                              v
              [ Session Registry (Redis) ]   [ Cassandra (messages) ]
                     |
                     v
              [ Recipient's Chat Server ] --> Recipient
                     |
              (if offline)
                     v
              [ Push Notification Service (APNs/FCM) ]
```

- **Chat Servers**: Stateful WebSocket servers, ~50K connections each. Handle message routing and delivery.
- **Redis**: Session registry (user to Chat Server mapping), per-conversation sequence counters, presence, typing indicators, and unread counts.
- **Cassandra**: Persistent message store, partitioned by conversation_id with sequence number as clustering key for ordered retrieval.
- **Push Notification Service**: Sends APNs/FCM notifications for offline users.

## 5. How a Message Is Sent and Ordered
1. Sender sends a message via WebSocket with a client-generated msg_id (for idempotency).
2. Chat Server atomically increments the conversation's sequence counter: `INCR conv_seq:{conv_id}` in Redis.
3. Message is stored in Cassandra with the assigned sequence number.
4. Chat Server ACKs to sender (message is now durable).
5. Chat Server looks up the recipient's connected Chat Server via Session Registry.
6. If online: route to recipient's Chat Server for WebSocket delivery. If offline: trigger push notification.
7. Recipient's client checks the sequence number. If there is a gap (e.g., received seq 4568 but not 4567), it fetches the missing message via REST API after a 5-second timeout.

## 6. What Happens When Things Fail?
- **Chat Server crashes**: 50K users disconnect and reconnect to another server. Messages already persisted in Cassandra are safe. Unacknowledged messages are retried by the sender (idempotent via msg_id).
- **Redis sequence counter fails**: Fall back to timestamp-based ordering until Redis recovers. Accept possible minor ordering issues during the brief failover window.
- **Cassandra goes down**: Messages cannot be persisted. Buffer in Kafka until Cassandra recovers. Only ACK to sender after Cassandra write completes to prevent data loss.

## 7. Scaling
- **10x (500M concurrent connections)**: 10,000+ Chat Servers with consistent hashing for user assignment. Tier old messages to cold storage (Parquet on S3 for messages older than 30 days). Redis INCR handles 10x easily since per-conversation rates stay constant.
- **100x**: Custom binary protocol replacing WebSocket for lower overhead. Purpose-built Chat Servers in C++/Rust for 500K connections per server. Multi-region with local sequence counters using region-prefixed sequences to avoid cross-region coordination.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Per-conversation sequence number vs global Snowflake IDs | Per-conversation gives total ordering within each conversation with no global bottleneck. Snowflake gives global ordering but does not help with per-conversation consistency. |
| Redis INCR for sequencing vs database-assigned sequences | Redis INCR is sub-millisecond and atomic. Database sequences add write latency. Redis failure is rare and falls back to timestamp ordering. |
| Group fan-out batched by Gateway Server (O(servers) vs O(members)) | For a 500-member group with 125 online across 5 servers, fan-out sends 5 messages instead of 125. Dramatically reduces network overhead for large groups. |

## 9. Closing (30s)
> "This chat system is built on three pillars: stateful WebSocket Chat Servers for real-time delivery, Cassandra for persistent message storage, and per-conversation sequence numbers in Redis for consistent ordering. Zero message loss is guaranteed through server-side persistence before ACK, client-side retry with idempotency keys, and push notifications for offline users. Group delivery is optimized from O(members) to O(gateway servers) by batching. Multi-device sync tracks the last received sequence per device, enabling efficient catch-up on reconnection."
