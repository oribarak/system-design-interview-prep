# System Design Interview: Chat System — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing a general-purpose chat system that supports one-on-one messaging, group chats, and real-time message delivery. This is a foundational design that underpins products like WhatsApp, Facebook Messenger, Discord, and in-app chat features. I will focus on the core messaging pipeline, connection management, message ordering and delivery guarantees, group chat architecture, and how we handle presence, notifications, and message synchronization across devices. I will make explicit trade-off decisions throughout, noting where a WhatsApp-style vs Slack-style system would diverge."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Is this a standalone chat app or an embeddable chat SDK for other applications?
  - *Assume: standalone chat application with mobile and web clients.*
- Do we need to support channels (Slack-like) or just 1:1 and group chats?
  - *Assume: 1:1 DMs and group chats (up to 500 members). No public channels.*

**Scale**
- How many users?
  - *Assume: 100M DAU, 50M concurrent connections.*
- Message volume?
  - *Assume: 10B messages/day.*

**Policy / Constraints**
- Message persistence?
  - *Assume: messages are persistent (stored permanently for chat history).*
- End-to-end encryption?
  - *Assume: optional, configurable per conversation. Default: encrypted in transit (TLS) + at rest (AES-256).*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 100M |
| Concurrent connections | 50M |
| Messages per day | 10B |
| Message send QPS | 10B / 86400 = ~116K QPS avg, ~300K peak |
| Average message size | 300 bytes (text + metadata) |
| Message storage per day | 10B * 300B = 3 TB/day |
| Message storage per year | ~1.1 PB/year |
| Media messages (15%) | 1.5B/day * 500KB avg = ~750 TB/day media |
| Chat servers needed | 50M connections / 50K per server = ~1,000 servers |
| Average group size | 15 members |
| Total conversations | ~5B (mostly 1:1) |

---

## 3) Requirements (3-5 min)

### Functional
- One-on-one messaging with text, images, video, files
- Group chat creation and messaging (up to 500 members)
- Message delivery with status indicators (sent, delivered, read)
- Online/offline presence indicators
- Message history retrieval with pagination
- Push notifications for offline users
- Multi-device sync (messages appear on all logged-in devices)
- Typing indicators

### Non-functional
- **Low latency**: message delivery in < 200ms for online recipients
- **Reliability**: zero message loss (at-least-once delivery)
- **Ordering**: messages within a conversation appear in consistent order for all participants
- **High availability**: 99.99% uptime
- **Scalability**: 100M DAU, 50M concurrent connections, 300K peak QPS
- **Consistency**: strong consistency for message ordering within a conversation, eventual for presence

---

## 4) API Design (2-4 min)

### WebSocket Events (bidirectional)

**Client -> Server:**
```json
// Send message
{ "type": "send", "msg_id": "uuid", "conv_id": "c123",
  "text": "hello", "media_ref": null }

// Typing indicator
{ "type": "typing", "conv_id": "c123", "is_typing": true }

// Mark read
{ "type": "read", "conv_id": "c123", "up_to_seq": 1234 }

// Presence heartbeat
{ "type": "ping" }
```

**Server -> Client:**
```json
// New message
{ "type": "message", "msg_id": "uuid", "conv_id": "c123",
  "sender_id": "u456", "text": "hello", "seq": 1234, "ts": 1678900000 }

// Delivery receipt
{ "type": "receipt", "msg_id": "uuid", "status": "delivered" }

// Typing indicator
{ "type": "typing", "conv_id": "c123", "user_id": "u456" }

// Presence update
{ "type": "presence", "user_id": "u789", "status": "online" }
```

### REST API (for history and management)

```
GET /v1/conversations/{conv_id}/messages?before_seq=<seq>&limit=50
POST /v1/conversations  (create group)
POST /v1/media/upload   (upload media, get media_ref)
GET /v1/conversations    (list user's conversations with last message)
```

---

## 5) Data Model (3-5 min)

### Users (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT PK | |
| username | VARCHAR(32) | unique |
| avatar_url | VARCHAR(512) | |
| last_seen | TIMESTAMP | |
| device_tokens | JSONB | push notification tokens |

### Conversations (PostgreSQL, sharded by conversation_id)
| Column | Type | Notes |
|---|---|---|
| conversation_id | BIGINT PK | |
| type | ENUM | dm, group |
| name | VARCHAR(128) | null for DMs |
| created_at | TIMESTAMP | |
| last_message_seq | BIGINT | monotonic counter |

### Messages (Cassandra, partitioned by conversation_id)
| Column | Type | Notes |
|---|---|---|
| conversation_id | BIGINT | partition key |
| seq | BIGINT | clustering key (ascending) |
| msg_id | UUID | client-generated, for idempotency |
| sender_id | BIGINT | |
| text | TEXT | |
| media_ref | VARCHAR | S3 path |
| created_at | TIMESTAMP | |

### Conversation Members (PostgreSQL)
| Column | Type | Notes |
|---|---|---|
| conversation_id | BIGINT | composite PK |
| user_id | BIGINT | composite PK |
| last_read_seq | BIGINT | for unread count |
| role | ENUM | owner, admin, member |
| muted_until | TIMESTAMP | null if not muted |

### User Conversations Index (Cassandra, partitioned by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT | partition key |
| last_message_ts | TIMESTAMP | clustering key (desc) |
| conversation_id | BIGINT | |
| last_message_preview | TEXT | |
| unread_count | INT | |

### Session Registry (Redis)
- Key: `session:{user_id}`
- Value: `{server_id, connected_at, device_id}`
- TTL: refreshed on heartbeat, expires on disconnect

**Storage choices**: Cassandra for messages (high write throughput, time-ordered retrieval by partition). PostgreSQL for conversations and memberships (relational queries, strong consistency). Redis for sessions, presence, typing indicators, and unread counts.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow** (1:1 message):
1. Sender's client sends message via WebSocket to their connected Chat Server
2. Chat Server assigns a monotonic sequence number from the conversation's sequence counter
3. Chat Server persists message to Cassandra
4. Chat Server looks up recipient's Chat Server via Session Registry
5. If recipient online: route to their Chat Server for WebSocket delivery
6. If recipient offline: enqueue for push notification and store as pending

**Components**:
- **Chat Servers**: stateful WebSocket servers, each handles ~50K connections
- **Session Registry**: Redis-based mapping of user_id to Chat Server (for routing)
- **Message Store**: Cassandra for persistent message storage
- **Sequence Generator**: per-conversation monotonic counter for message ordering
- **Conversation Service**: manages conversations, memberships, settings
- **Presence Service**: tracks online/offline/away status
- **Push Service**: sends push notifications via APNs/FCM for offline users
- **Sync Service**: handles multi-device message synchronization
- **Media Service**: file upload, processing, storage in S3
- **CDN**: media delivery
- **Kafka**: event bus for analytics, push notification triggers, multi-device sync
- **Redis Cluster**: sessions, presence, typing, unread counts, sequence counters

### Routing: How Does WS1 Find WS2?

With dozens (or thousands) of WebSocket servers, when a sender's Chat Server (WS1) needs to deliver a message to a recipient on a different Chat Server (WS2), the routing problem must be solved:

**Option A — Session Registry lookup (chosen for baseline):** Redis stores `session:{user_id} -> {server_id, device_id}`. WS1 looks up the recipient's server and routes directly. Simple, but requires maintaining accurate session state.

**Option B — Redis Pub/Sub (preferred for simplicity at scale):** Each Chat Server subscribes to channels for its connected users: `user:{user_id}`. When WS1 needs to deliver a message, it publishes to `user:{recipient_id}`. The Chat Server with that user's connection receives the message and delivers it via WebSocket. This eliminates the need for a separate Session Registry lookup on the send path — the Pub/Sub subscription implicitly handles routing. The Session Registry is still useful for checking online status and for push notification decisions.

### Canonical 1:1 Conversation ID

For DMs, ensure a unique conversation per user pair by enforcing a canonical ordering: `participant_1 = MIN(user_a, user_b)`, `participant_2 = MAX(user_a, user_b)`. Create a unique index on `(participant_1, participant_2)`. This prevents duplicate conversations between the same two users regardless of who initiates.

**One-sentence summary**: A stateful WebSocket architecture with Chat Servers handling persistent connections, Cassandra for message persistence with per-conversation sequence ordering, and Redis for session routing (via Pub/Sub or direct lookup) and real-time state management.

---

## 7) Deep Dive #1: Message Ordering and Sequence Numbers (8-12 min)

Message ordering is the most critical consistency guarantee in a chat system. All participants must see messages in the same order.

### The Problem with Timestamps
- Clients have different clocks (skew can be seconds or even minutes)
- Two clients sending simultaneously could produce conflicting orderings
- Server timestamps solve client skew but introduce issues with network latency (message that was sent first may arrive at server second)

### Solution: Per-Conversation Sequence Numbers

Each conversation maintains a monotonic sequence counter:
```
conversation_id: c123
last_message_seq: 4567
```

When a new message arrives:
1. Chat Server atomically increments the conversation's sequence counter: `seq = INCR conv_seq:{conv_id}` (Redis atomic operation)
2. The message is stored with this sequence number
3. The sequence number is the definitive ordering -- not the timestamp
4. Timestamp is stored for display purposes only

### Sequence Generator Architecture
- **Redis INCR**: atomic, single-threaded. Perfect for per-conversation counters.
- Each conversation has its own counter: `conv_seq:{conversation_id}`
- Redis cluster sharded by conversation_id
- If Redis fails: fall back to database sequence (PostgreSQL `nextval()`)

### Why Per-Conversation (not Global)?
- Global sequence (like Twitter's Snowflake) doesn't help -- we need ordering within a conversation, not across conversations
- Per-conversation counter is simpler and avoids single global bottleneck
- Each conversation generates at most ~100 messages/minute (99th percentile), well within Redis INCR capacity

### Concurrent Messages in Same Conversation
If two users send messages simultaneously:
1. Both messages arrive at (potentially different) Chat Servers
2. Both Chat Servers INCR the same Redis key `conv_seq:{conv_id}`
3. Redis serializes the INCRs -- one gets seq=4568, other gets seq=4569
4. Both messages are stored with their assigned sequences
5. All clients display messages in sequence order, regardless of which arrived first

### Gap Handling
What if seq 4568 is persisted but seq 4567 is delayed?
- Client tracks the latest sequence it has seen
- If a gap is detected (received 4568 but not 4567), client requests missing messages via REST API
- Missing message is likely in-flight and will arrive shortly
- After 5 seconds of gap: fetch via REST to fill the hole

### Offline Delivery: Inbox + In-Flight Pattern

When a recipient is offline, messages must be reliably queued and delivered when they come back online. The inbox/in-flight pattern guarantees at-least-once delivery without message loss:

1. **Inbox**: When the recipient is offline, the message is written to the user's inbox (a per-user queue in Redis or Cassandra).
2. **In-flight**: When the user reconnects, messages are moved from inbox to an in-flight set. They are delivered over WebSocket.
3. **ACK**: The client acknowledges receipt of each message (or batch). On ACK, the message is removed from in-flight.
4. **Reaper**: A background process scans the in-flight set for stale items (e.g., delivered > 60 seconds ago with no ACK). Stale items are moved back to inbox for redelivery.

This mirrors the SQS visibility-timeout pattern: messages are never deleted until explicitly acknowledged, and a reaper handles the case where delivery succeeds but ACK is lost.

### Multi-Device Delivery

For users logged into multiple devices, each device needs independent delivery tracking:

- **Per-device inbox**: Each device has its own delivery queue. When a message arrives for a user, it is fanned out to all of that user's device inboxes.
- **Per-device delivery status**: Each device independently ACKs messages. A message is considered "delivered" to a user only when all active devices have ACKed.
- **Device sync on reconnect**: Each device tracks its own `last_received_seq` per conversation. On reconnection, it requests messages since its last known seq: `GET /messages?after_seq=4500`.
- This handles the "new device" case: sync all messages from seq=0 (with pagination).

---

## 8) Deep Dive #2: Group Chat Delivery (5-8 min)

Group messages introduce a fan-out challenge: one message must be delivered to up to 500 members.

### Delivery Architecture

1. Sender posts message to group conversation
2. Chat Server assigns sequence number and persists the message (one copy)
3. Chat Server looks up group members from the membership table (cached in Redis)
4. For each member:
   - Query Session Registry: is this member online?
   - If online: route message to their Chat Server
   - If offline: increment their unread count, trigger push notification (if not muted)

### Optimization: Batched Gateway Delivery
Instead of routing N individual messages to N Chat Servers:
1. Group member sessions by their Chat Server: "Gateway A has members [u1, u3, u7], Gateway B has [u2, u4]..."
2. Send one message per Gateway Server with the list of recipient user_ids
3. Each Gateway Server locally delivers to its connected users
4. Reduces network calls from O(members) to O(gateway_servers)

### Read Receipts: High-Water Mark Pattern

Rather than tracking read status per-message (which explodes storage), use a **high-water mark** approach:

- Store `last_read_sequence_number` per user per conversation (already in `conversation_members.last_read_seq`).
- All messages with `seq <= last_read_seq` are considered read. No per-message read tracking needed.
- When the client scrolls through messages, it sends a `read` event: `{ "type": "read", "conv_id": "c123", "up_to_seq": 1234 }`.
- **Client-side debounce (2 seconds)**: As the user scrolls through many unread messages, don't send a `read` event for every message. Wait 2 seconds after the user stops scrolling, then send a single `read` event with the highest visible sequence number. This dramatically reduces write traffic.
- Unread count = `conversation.last_message_seq - member.last_read_seq`.

### Delivery Receipt in Groups
- Tracking per-member delivery/read status for 500 members per message is expensive
- Storage: `{msg_seq: {user_1: delivered, user_2: read, ...}}` -- 500 entries per message
- Optimization: only track for the sender's view (they see who has read)
- For groups > 100 members: show approximate read count, not individual status

### Group Member Changes
- When a member is added: they can see messages from the point of joining (not historical)
- When a member is removed: they retain their local message history but receive no new messages
- Session Registry is updated immediately on member changes

### Large Group Performance
For a group with 500 members:
- ~125 online at any time (25%)
- Spread across ~3-5 Chat Servers
- Fan-out latency: < 10ms (parallel dispatch to 3-5 servers)
- Push notifications for ~375 offline members: batched, sent over 5 seconds

---

## 9) Trade-offs and Alternatives (3-5 min)

### Sequence Numbers vs Vector Clocks vs Lamport Timestamps
| Approach | Pros | Cons |
|---|---|---|
| Per-conversation sequence (chosen) | Simple, total ordering, easy gap detection | Single point of serialization (Redis) |
| Vector clocks | True causal ordering, no single point | Complex for clients, O(n) per message |
| Lamport timestamps | Logical ordering, distributed | Doesn't capture causality, partial ordering |

Per-conversation sequence numbers are the right trade-off: simple, total ordering, and the serialization point (Redis INCR) handles the load easily at per-conversation message rates.

### Cassandra vs MySQL/PostgreSQL for Messages
| Aspect | Cassandra | MySQL/PostgreSQL |
|---|---|---|
| Write throughput | Excellent | Good (with sharding) |
| Partition-based queries | Native | Requires sharding setup |
| Consistency | Tunable | Strong |
| Operational complexity | Higher | Lower (with Vitess) |

Cassandra is ideal for the append-only, partition-key-based message access pattern. MySQL (with Vitess) is a viable alternative and is what Discord actually uses.

### Polling vs WebSocket vs SSE
| Aspect | WebSocket | SSE | Long Polling |
|---|---|---|---|
| Latency | Lowest | Low | Moderate |
| Bidirectional | Yes | Server -> Client only | Simulated |
| Connection overhead | 1 per user | 1 per user | New conn each poll |
| Best for | Real-time chat | Notifications | Simple clients |

WebSocket is the clear choice for a chat system: both parties need to push data (sender pushes messages, server pushes incoming messages, typing indicators, presence updates). SSE is server-to-client only, so the client would still need a separate channel to send messages. HTTP polling adds latency and connection overhead.

### Push vs Pull for Offline Messages
- **Push on connect** (chosen): when a user connects, the server pushes all pending messages. Simple, fast delivery.
- **Pull on demand**: client requests messages per conversation. More bandwidth-efficient but higher latency.
- Our approach: push a summary (unread counts + last message preview per conversation) on connect, then pull full messages when the user opens a conversation.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (1B DAU, 500M concurrent connections)
- **Chat Servers**: 10,000+ servers. Use consistent hashing per user_id for server assignment.
- **Session Registry**: Redis cluster with read replicas. Hot users (in many groups) may cause hot keys -- partition session data by user_id.
- **Message storage**: 11 PB/year. Tier old messages to cold storage (S3 with Parquet format). Recent messages (< 30 days) in Cassandra, older via REST from cold storage.
- **Sequence generators**: Redis INCR handles 10x easily (each conversation's rate stays constant). Total Redis load grows with new conversations, scale cluster horizontally.

### At 100x
- **Custom protocol**: replace WebSocket with a custom binary protocol for lower overhead per connection
- **Custom connection servers**: purpose-built servers in C++/Rust for higher connection density (500K per server instead of 50K)
- **Multi-region with local sequencing**: each region assigns sequences independently with region prefix to avoid cross-region coordination
- **Message compaction**: for conversations with very long history, implement message compaction (merge old messages into summary blocks)

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Chat Server crash**: 50K users disconnect. Client reconnects to another server via load balancer. Session Registry updates. Messages in-flight during crash are safe because they were persisted to Cassandra before delivery.
- **Redis (session registry) failure**: cannot route messages. Fall back to broadcast (send to all Chat Servers, each checks if user is connected). Expensive but functional.
- **Cassandra failure**: messages cannot be persisted. Buffer in Kafka until Cassandra recovers. Risk: Chat Server acknowledges message before persistence. Mitigation: only ACK after Cassandra write completes.
- **Sequence counter (Redis) failure**: fall back to timestamp-based ordering until Redis recovers. Accept possible minor ordering issues during failover.

**Zero message loss guarantee**:
1. Client sends message with client-generated msg_id
2. Chat Server persists to Cassandra
3. Chat Server returns ACK to client
4. If no ACK within 5 seconds: client retries
5. Server deduplicates by msg_id
6. Result: at-least-once delivery + client-side dedup = exactly-once display

**Multi-region**:
- Users connect to nearest region
- Cross-region messages route through backbone
- Each conversation's sequence counter is in one primary region (the conversation's home region)
- Cross-region message send adds ~30-50ms latency for the sequence assignment round-trip

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- End-to-end message delivery latency: p50, p95, p99
- WebSocket connection count and reconnection rate
- Sequence counter latency (Redis INCR)
- Message persistence latency (Cassandra write)
- Push notification delivery rate and latency
- Unread count accuracy
- Group message fanout time

**Alerting**:
- Delivery p99 > 500ms -- page on-call
- Connection count per Chat Server > 55K (110% capacity) -- auto-scale
- Redis INCR latency > 1ms -- warn
- Cassandra write errors > 0.1% -- critical
- Push notification failure rate > 5% -- warn

**Dashboards**:
- Message delivery funnel: sent -> persisted -> routed -> delivered -> read
- Connection density per Chat Server
- Group fanout performance by group size
- Cross-region message latency heatmap

---

## 13) Security (1-3 min)

- **Authentication**: JWT-based with short-lived tokens. Device registration with token refresh.
- **Authorization**: conversation membership enforced on every message send and receive. Users cannot send to conversations they are not members of.
- **Encryption**: TLS 1.3 in transit. AES-256 at rest. Optional E2E encryption per conversation (Signal Protocol integration).
- **Rate limiting**: max 60 messages/minute per user. Prevents spam and abuse.
- **Content moderation**: optional server-side content scanning (only for non-E2E conversations). ML-based spam/abuse detection.
- **Account security**: multi-factor authentication, login anomaly detection.
- **Media security**: uploaded files scanned for malware. Pre-signed URLs with short TTL for downloads.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Messaging Core team: message pipeline, delivery guarantees, ordering
- Connection Infrastructure team: Chat Servers, WebSocket management, session routing
- Storage team: Cassandra operations, message archival, data lifecycle
- Notifications team: push notifications, presence, typing indicators
- Client teams: iOS, Android, Web, Desktop

**Deployment**:
- Chat Server updates: rolling restart with connection draining
  - Drain phase: stop accepting new connections, wait for existing ones to migrate (5 min timeout)
  - Restart with new code
  - Clients auto-reconnect to other servers during drain
- Database schema changes: online migration with shadow tables
- Feature flags for all new features

**On-call**: 24/7 rotation. Messaging outages are highly visible. Separate escalation paths for connection issues vs delivery issues vs storage issues.

---

## 15) Common Follow-up Questions

**Q: How do you handle the conversation list screen (inbox)?**
A: The User Conversations Index (Cassandra, partitioned by user_id, ordered by last_message_ts descending) provides the inbox view. Each entry has the conversation_id, last message preview, unread count, and timestamp. When a new message arrives in any conversation, the corresponding entry in the user's index is updated. This denormalization avoids expensive joins at read time.

**Q: How do you handle message deletion?**
A: Soft delete: set a `deleted_at` timestamp on the message. All clients are notified via the same fanout mechanism and replace the message with "This message was deleted." The message content is removed from the search index. For compliance-sensitive deployments, hard delete the content but retain metadata.

**Q: How do you handle very long conversations (millions of messages)?**
A: Cassandra partitioned by conversation_id handles large partitions well, but at millions of messages, partition size grows large. Solution: compound partition key `(conversation_id, bucket)` where bucket = `seq / 10000`. Each bucket contains at most 10K messages. Client paginates across buckets transparently.

**Q: How would you add end-to-end encryption to this system?**
A: Layer the Signal Protocol on top. Key distribution service manages identity keys and pre-keys. Messages are encrypted at the client before sending and decrypted at the recipient's client. The server stores and routes encrypted blobs. Trade-off: server-side search is no longer possible for E2E conversations.

**Q: How do you handle @mentions in group chats?**
A: The message payload includes a `mentions` array of user_ids. The Notification Service checks this array and sends push notifications to mentioned users even if they have muted the group. The client renders @mentions with highlighting by matching user_ids in the mentions array to display names.

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed a general-purpose chat system built on three pillars: stateful WebSocket Chat Servers for real-time delivery, Cassandra for persistent message storage, and per-conversation sequence numbers in Redis for consistent ordering. The sequence number approach is simple yet powerful -- Redis INCR provides atomic serialization within each conversation, clients detect gaps and self-heal, and multi-device sync is achieved by tracking the last received sequence per device. Group chat delivery is optimized from O(members) to O(gateway servers) by batching messages per Chat Server. The system guarantees zero message loss through server-side persistence before ACK, client-side retry with idempotency keys, and push notifications for offline users. The architecture cleanly separates concerns: real-time delivery (WebSocket + Redis), persistence (Cassandra), and notifications (push service), allowing each to scale and fail independently."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `max_connections_per_server` | 50,000 | WebSocket connections per Chat Server |
| `max_group_size` | 500 | Maximum group chat members |
| `message_retry_timeout` | 5s | Client retry timeout for unacknowledged messages |
| `sequence_bucket_size` | 10,000 | Messages per Cassandra partition bucket |
| `presence_heartbeat_interval` | 30s | Client heartbeat for online status |
| `typing_indicator_ttl` | 5s | Auto-expire typing indicator |
| `push_notification_batch_size` | 100 | Group notification batch size |
| `message_history_page_size` | 50 | Messages per history request |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Per-conversation sequence number** | Monotonically increasing counter per conversation, providing total message ordering |
| **Chat Server** | Stateful server maintaining persistent WebSocket connections with clients |
| **Session Registry** | Redis mapping of user_id to their connected Chat Server for message routing |
| **Fan-out** | Delivering a group message to all members by routing to their respective Chat Servers |
| **At-least-once delivery** | Guarantee that a message will be delivered one or more times (deduped at client) |
| **Connection draining** | Graceful shutdown process where a server stops accepting new connections before restarting |
| **Gap detection** | Client detecting missing sequence numbers and fetching them via REST API |
| **Idempotency key** | Client-generated msg_id used to deduplicate retried messages on the server |

## Appendix C: References

- [14_whatsapp](/14_whatsapp) -- E2E encrypted messaging variant
- [15_slack](/15_slack) -- channel-based team messaging variant
- [17_websocket_notification_system](/17_websocket_notification_system) -- WebSocket infrastructure design
- [19_push_notifications](/19_push_notifications) -- push notification system
- [22_key_value_store](/22_key_value_store) -- Redis as session and sequence store
