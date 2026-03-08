# System Design Interview: WhatsApp — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing WhatsApp -- a real-time messaging platform supporting one-on-one chats, group chats, media sharing, and end-to-end encryption for 2 billion users. The core challenges are reliable message delivery with ordering guarantees, end-to-end encryption that prevents even the server from reading messages, presence and typing indicators, and group messaging at scale. I will focus on the messaging infrastructure, the delivery guarantee system, encryption architecture, and how we handle offline users and message synchronization across devices."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing text messaging only, or also voice/video calls?
  - *Assume: text and media messaging, group chats. Voice/video calls mentioned but not deep-dived.*
- Do we need multi-device support (like WhatsApp Web)?
  - *Assume: yes, messages must sync across phone and companion devices.*

**Scale**
- How many users and messages?
  - *Assume: 2B registered users, 500M DAU, 100B messages/day.*
- Group chat limits?
  - *Assume: max 1,024 members per group.*

**Policy / Constraints**
- Is end-to-end encryption a hard requirement?
  - *Assume: yes, the server must never have access to plaintext messages.*
- Message retention policy?
  - *Assume: messages are stored on the server only until delivered. Once all recipients receive the message, the server deletes it.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 500M |
| Messages per day | 100B |
| Message send QPS | 100B / 86400 = ~1.16M QPS avg, ~3M peak |
| Average message size | 200 bytes (text), 500KB (media) |
| Text message bandwidth | 1.16M * 200 bytes = ~232 MB/s |
| Media messages (20% of total) | 20B * 500KB = ~10 PB/day |
| Concurrent WebSocket connections | ~200M (40% of DAU online at any time) |
| Messages per group (avg) | 50 members, 100 messages/day/group |
| Total groups | ~500M |
| Undelivered message storage | ~2 TB (messages waiting for offline users) |

The system is write-heavy and connection-heavy. 200M concurrent WebSocket connections is the primary infrastructure challenge.

---

## 3) Requirements (3-5 min)

### Functional
- One-on-one messaging with text, images, video, documents, voice messages
- Group messaging (up to 1,024 members)
- End-to-end encryption for all messages
- Message delivery status: sent, delivered, read (blue ticks)
- Typing indicators and online/last-seen presence
- Offline message queuing: messages delivered when recipient comes online
- Multi-device sync (phone + web/desktop companion devices)

### Non-functional
- **Low latency**: message delivery in < 200ms for online users
- **Reliability**: zero message loss (at-least-once delivery, exactly-once display)
- **Ordering**: messages within a conversation maintain sender's ordering
- **High availability**: 99.99% uptime
- **Scalability**: 200M concurrent connections, 3M QPS peak
- **Security**: end-to-end encryption, server has zero knowledge of message content

---

## 4) API Design (2-4 min)

### WebSocket Connection
```
WS /v1/ws
Headers: Authorization: Bearer <token>
On connect: server sends pending messages and presence updates
```

### Send Message
```
WebSocket frame:
{
  type: "message",
  msg_id: "<client-generated UUID>",
  conversation_id: "<chat_id>",
  recipient_id: "<user_id>" | null (group),
  encrypted_payload: "<base64>",
  media_ref?: "<s3_key>",
  timestamp: "<client_timestamp>"
}
```

### Delivery Receipt
```
WebSocket frame:
{
  type: "receipt",
  msg_id: "<msg_id>",
  status: "delivered" | "read",
  timestamp: "<timestamp>"
}
```

### Get Conversation History (multi-device sync)
```
GET /v1/conversations/{conversation_id}/messages?since=<timestamp>&limit=50
Response: 200 { messages: EncryptedMessage[], next_cursor }
```

### Upload Media
```
POST /v1/media/upload
Body: encrypted media file
Response: 200 { media_id, media_url }
```

---

## 5) Data Model (3-5 min)

### Users Table (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT PK | phone number hash |
| phone_hash | VARCHAR | for contact discovery |
| public_key | BYTEA | E2E encryption identity key |
| device_tokens | JSONB | push notification tokens |
| last_seen | TIMESTAMP | |

### Messages Table (Cassandra, partitioned by conversation_id)
| Column | Type | Notes |
|---|---|---|
| conversation_id | BIGINT | partition key |
| message_id | TIMEUUID | clustering key (time-ordered) |
| sender_id | BIGINT | |
| encrypted_payload | BLOB | E2E encrypted content |
| media_ref | VARCHAR | S3 key for media |
| status | MAP<BIGINT, VARCHAR> | per-recipient delivery status |
| created_at | TIMESTAMP | |

### Conversations Table (PostgreSQL, sharded by user_id)
| Column | Type | Notes |
|---|---|---|
| user_id | BIGINT | shard key |
| conversation_id | BIGINT | |
| type | ENUM | direct, group |
| last_message_at | TIMESTAMP | |
| unread_count | INT | |

### Group Members Table (PostgreSQL)
| Column | Type | Notes |
|---|---|---|
| group_id | BIGINT | composite PK |
| member_id | BIGINT | composite PK |
| role | ENUM | admin, member |
| joined_at | TIMESTAMP | |

### Offline Message Queue (Redis sorted set)
- Key: `pending:{user_id}`
- Score: timestamp
- Value: serialized message
- Messages deleted after delivery confirmation

**Storage choices**: Messages in Cassandra for high write throughput and time-ordered retrieval. Conversations in PostgreSQL for relational queries. Redis for offline message queuing (fast push/pop). Media in S3 (encrypted at client before upload).

---

## 6) High-Level Architecture (5-8 min)

**Dataflow** (one-on-one message):
1. Sender's device encrypts message with recipient's public key
2. Sender sends encrypted message via WebSocket to Chat Server
3. Chat Server looks up recipient's connected Chat Server via Session Service
4. If online: route to recipient's Chat Server, deliver via WebSocket
5. If offline: store in offline message queue, send push notification
6. When recipient connects, flush offline queue

**Components**:
- **Chat Servers**: stateful WebSocket servers, each handles ~50K connections
- **Session Service**: maps user_id to their connected Chat Server (Redis)
- **Message Router**: routes messages between Chat Servers
- **Offline Queue**: stores messages for offline users (Redis + overflow to Cassandra)
- **Message Store**: persistent storage for message history (Cassandra)
- **Presence Service**: tracks online/offline status, typing indicators
- **Group Service**: manages group membership, handles group message fan-out
- **Push Notification Service**: sends APNs/FCM notifications for offline users
- **Key Distribution Service**: manages E2E encryption key exchange
- **Media Service**: handles encrypted media upload/download via S3
- **CDN**: not used for message content (encrypted, can't cache), used for profile photos
- **Kafka**: event bus for delivery receipts, analytics

**One-sentence summary**: A stateful WebSocket architecture where Chat Servers maintain persistent connections, a Session Service routes messages between servers, and an offline queue ensures delivery to disconnected users, all protected by end-to-end encryption.

---

## 7) Deep Dive #1: Message Delivery Guarantees (8-12 min)

Reliable message delivery is the core promise. No message can ever be lost.

### Delivery Flow with Guarantees

**Step 1: Client to Server (at-least-once)**
- Client sends message via WebSocket with a client-generated `msg_id`
- Server persists message and returns ACK with `msg_id`
- If client doesn't receive ACK within 5 seconds, it retries
- Server uses `msg_id` for idempotency -- duplicate sends are detected and ignored

**Step 2: Server to Recipient (at-least-once)**
- If recipient online: deliver via WebSocket, wait for device ACK
- If no ACK in 3 seconds: retry delivery
- After 3 retries: mark as pending, deliver on next connection
- If recipient offline: store in offline queue immediately

**Step 3: Offline Queue Flush**
- When user connects, Chat Server queries `pending:{user_id}` in Redis
- Delivers all pending messages in order
- Waits for ACK for each batch before deleting from queue
- If connection drops during flush, remaining messages stay in queue

### Ordering Guarantees
- Messages within a conversation are ordered by the sender's timestamp
- Server assigns a server-side sequence number per conversation for tie-breaking
- Cassandra clustering key (TIMEUUID) ensures ordered storage and retrieval
- Client maintains a local sequence counter to detect out-of-order delivery and reorder

### Exactly-Once Display
- Messages may be delivered multiple times (at-least-once)
- Client deduplicates by `msg_id` before displaying
- Client maintains a set of recently received `msg_id`s (last 1000)
- This gives at-least-once delivery + client-side dedup = exactly-once display

### Delivery Status Pipeline
```
SENT:       sender -> server ACK
DELIVERED:  server -> recipient device ACK
READ:       recipient opens conversation -> read receipt sent back
```

Each status update is a lightweight message routed back to the sender through the same WebSocket infrastructure.

### Edge Cases
- **Both users offline**: message stored in sender's outbox (client-side) and retried when sender comes online. Server stores in offline queue for recipient.
- **Group message partial delivery**: track per-member delivery status. If 48/50 members receive it, the 2 remaining get it from offline queue later. Delivery ticks show: one tick (sent), two ticks (delivered to all), blue ticks (read by all).
- **Device switch during delivery**: session service updates routing. In-flight messages to old server are NACKed and redelivered to new server.

---

## 8) Deep Dive #2: End-to-End Encryption (5-8 min)

### Signal Protocol (used by WhatsApp)
WhatsApp uses the Signal Protocol for E2E encryption:

**Key Exchange**:
1. Each device generates an identity key pair (long-term) and a set of pre-keys (ephemeral)
2. Pre-keys are uploaded to the Key Distribution Service
3. When User A wants to message User B for the first time:
   - A fetches B's public identity key + one pre-key from the server
   - A performs X3DH (Extended Triple Diffie-Hellman) to establish a shared secret
   - This shared secret seeds a Double Ratchet for forward secrecy

**Double Ratchet**:
- Each message uses a unique encryption key derived from the ratchet
- After each message, the ratchet advances -- old keys are deleted
- Compromising one key doesn't reveal past or future messages (forward secrecy + post-compromise security)

**Server's Role**:
- Server stores and routes encrypted blobs
- Server distributes public keys (but cannot decrypt)
- Server never sees plaintext content

### Group Encryption
- **Sender Key protocol**: each member generates a sender key
- Sender encrypts message once with their sender key
- Sender key is distributed to all group members via pairwise E2E channels
- When a member is removed, all sender keys are rotated (new keys distributed)
- This avoids encrypting the message N times for N members

### Media Encryption
1. Sender encrypts media with a random AES-256 key
2. Encrypted media blob is uploaded to S3
3. AES key is sent inside the E2E encrypted message payload
4. Recipient downloads encrypted blob from S3, decrypts with the key from the message

**Trade-offs**:
- Server cannot scan content for spam/abuse (privacy vs safety trade-off)
- Key rotation on group member removal is expensive for large groups (1,024 members)
- Multi-device: each device has its own identity key. Messages must be encrypted separately for each device (up to 4 devices per user)

---

## 9) Trade-offs and Alternatives (3-5 min)

### WebSocket vs Long Polling vs HTTP/2 Server Push
| Aspect | WebSocket | Long Polling | HTTP/2 Server Push |
|---|---|---|---|
| Latency | Very low (persistent conn) | Moderate (new request each time) | Low |
| Server resources | High (1 conn per user) | Moderate | Moderate |
| Battery (mobile) | Good (single conn) | Bad (repeated connections) | Good |
| Scalability | 200M connections = ~4,000 servers | Fewer connections | Moderate |

WebSocket is essential for real-time chat. The connection overhead is justified by the latency requirement.

### Message Storage: Store-and-Forward vs Store-Permanently
- **Store-and-forward** (WhatsApp's approach): server stores messages only until delivered, then deletes. Privacy-preserving, lower storage cost.
- **Store permanently** (Telegram/iMessage): server retains all messages for sync across devices. Better multi-device experience, higher storage cost.
- WhatsApp compromises: messages are stored temporarily for offline delivery (up to 30 days), then deleted. Multi-device sync uses device-to-device transfer for history.

### Push vs Pull for Offline Messages
- Push (on connect): server pushes all pending messages as soon as user connects. Simple, fast.
- Pull (on demand): client requests messages for each conversation. More control, but higher latency.
- WhatsApp uses push: on connection, server flushes all pending messages immediately.

### Cassandra vs MongoDB vs PostgreSQL for Messages
| Aspect | Cassandra | MongoDB | PostgreSQL |
|---|---|---|---|
| Write throughput | Excellent | Good | Moderate |
| Time-ordered queries | Native (TIMEUUID) | Needs index | Needs index |
| Scalability | Linear | Good with sharding | Complex sharding |
| Operational complexity | Moderate | Moderate | Lower |

Cassandra wins for the message workload: high write throughput, natural time ordering, linear scalability.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (30M QPS, 2B concurrent connections)
- **Chat Servers**: 40,000+ servers at 50K connections each. Use consistent hashing for user-to-server assignment.
- **Session Service**: Redis cluster becomes huge. Shard by user_id, use local caching on Chat Servers for hot sessions.
- **Offline queue**: Redis may not fit all pending messages. Overflow to Cassandra for users offline > 1 hour.
- **Group fan-out**: 1,024-member groups with 30M QPS creates hot partitions. Shard group message delivery across multiple workers.

### At 100x
- **Custom protocol**: replace WebSocket with a custom binary protocol for lower overhead (less framing, smaller headers). WhatsApp actually uses a custom XMPP variant.
- **Custom infrastructure**: build purpose-built Chat Server hardware with high connection density (Erlang/BEAM VM, which WhatsApp originally used, handles millions of connections per server).
- **Edge computing**: deploy Chat Servers at edge PoPs for lower latency. Regional message routing.
- **Compression**: protocol-level message compression (dictionary-based for repeated patterns in chat).

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Chat Server crash**: user's WebSocket disconnects. Client reconnects to a different server. Session Service updates routing. Pending messages in the crashed server's memory are lost -- but they were already persisted to the offline queue before being sent, so they can be re-delivered.
- **Session Service (Redis) failure**: message routing fails. Fall back to broadcasting to all Chat Servers (expensive but functional). Rebuild session state as users reconnect.
- **Cassandra failure**: message persistence temporarily unavailable. Queue messages in Kafka until Cassandra recovers. No data loss due to Kafka buffering.
- **Push notification failure**: retry with exponential backoff. Multiple provider fallback (APNs primary, FCM backup for iOS users).

**Message durability guarantee**:
1. Client sends message -> server persists to Cassandra + offline queue -> server ACKs
2. If server crashes before persisting: client doesn't receive ACK, retries
3. If server crashes after persisting but before ACK: client retries, server deduplicates by msg_id
4. Result: zero message loss

**Multi-region**:
- Users connect to the nearest region's Chat Server
- Cross-region messages route through backbone network
- Offline queues are region-local (user's home region)

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Message delivery latency (sender to recipient device ACK): p50, p99
- WebSocket connection count per Chat Server
- Offline queue depth (total pending messages)
- Message delivery success rate
- Encryption key exchange latency
- Group message fan-out time

**Alerting**:
- Delivery p99 > 500ms -- warn
- Delivery p99 > 2s -- page on-call
- Offline queue growing > 10% per hour -- investigate (users not coming online or delivery stuck)
- Chat Server connections > 60K (80% capacity) -- pre-scale
- Push notification failure rate > 5% -- page on-call

**Dashboards**:
- Real-time message throughput and latency heatmap by region
- Connection density per Chat Server
- Offline queue depth over time
- Delivery funnel: sent -> persisted -> routed -> delivered -> read

---

## 13) Security (1-3 min)

- **End-to-end encryption**: Signal Protocol ensures only sender and recipient can read messages. Server is zero-knowledge.
- **Key verification**: users can verify each other's identity keys via QR code or safety number comparison.
- **Transport security**: TLS 1.3 for all client-server communication (double-encrypted: TLS + E2E).
- **Authentication**: phone number verification via SMS/voice OTP. Device-bound registration.
- **Abuse prevention**: client-side reporting (user forwards reported message to moderation). Metadata analysis (message frequency, group creation rate) for spam detection without reading content.
- **Account security**: two-step verification (PIN), registration lock.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Messaging Infrastructure team: Chat Servers, message routing, delivery guarantees
- Encryption team: Signal Protocol implementation, key management
- Group Chat team: group messaging, membership, administration
- Reliability team: connection management, offline queues, push notifications
- Client team: mobile apps, multi-device sync

**Deployment**:
- Chat Server updates: rolling restart with connection draining (gracefully close connections, let clients reconnect to updated servers)
- Protocol changes: backward-compatible versioning in message framing
- Zero-downtime deployments are critical -- any restart disrupts active conversations

**On-call**: 24/7 rotation with regional coverage. Messaging outages are highly visible and escalated immediately.

---

## 15) Common Follow-up Questions

**Q: How does multi-device sync work with E2E encryption?**
A: Each device has its own identity key pair. When sending a message, the sender encrypts it separately for each of the recipient's devices (up to 4). Companion devices (Web, Desktop) are linked to the primary phone via QR code pairing. Message history is transferred directly from phone to companion device (encrypted device-to-device), not through the server.

**Q: How do you handle a user who has been offline for 30 days?**
A: Messages are stored in the offline queue for up to 30 days. After 30 days, undelivered messages are permanently deleted. When the user reconnects, they receive all messages from the last 30 days. The sender sees a single check (sent but not delivered). If the message expires, the sender is not notified -- the message simply never gets a second check.

**Q: How do read receipts work in group chats?**
A: Each member's read status is tracked independently. The double blue ticks appear only when ALL members have read the message. The group owner or sender can tap the message to see per-member delivery and read timestamps. This requires storing a status map: `{member_id: status}` per message.

**Q: How do you prevent spam in WhatsApp?**
A: Without reading content (E2E encrypted), WhatsApp uses behavioral signals: message frequency, number of unique recipients, account age, group creation rate, and user reports. Forwarded messages are labeled and rate-limited (max 5 forwards). Accounts exhibiting spam patterns are banned.

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed WhatsApp as a stateful WebSocket messaging system with end-to-end encryption. Chat Servers maintain persistent connections with ~50K users each, and a Session Service routes messages between servers. The delivery guarantee system ensures zero message loss through server-side persistence, client ACKs with retry, and idempotent deduplication. Messages are stored in Cassandra only until delivered, then deleted -- respecting user privacy. End-to-end encryption uses the Signal Protocol with the Double Ratchet for forward secrecy, and group chats use Sender Keys to avoid per-member encryption overhead. The offline queue in Redis ensures messages reach users who were disconnected, with push notifications via APNs/FCM. The system scales through horizontal Chat Server addition and consistent hashing for connection assignment."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `max_connections_per_server` | 50,000 | WebSocket connections per Chat Server |
| `message_retry_count` | 3 | Delivery retries before queuing as offline |
| `message_retry_timeout` | 5s | Time to wait for delivery ACK |
| `offline_queue_ttl_days` | 30 | Max days to retain undelivered messages |
| `max_group_size` | 1,024 | Maximum members per group |
| `media_max_size_mb` | 100 | Max media upload size |
| `typing_indicator_timeout` | 5s | How long to show typing indicator |
| `presence_heartbeat_interval` | 30s | Frequency of online status heartbeat |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Signal Protocol** | Cryptographic protocol providing E2E encryption with forward secrecy and post-compromise security |
| **Double Ratchet** | Key derivation mechanism that generates unique keys per message and advances after each use |
| **X3DH** | Extended Triple Diffie-Hellman -- key agreement protocol for establishing shared secrets |
| **Sender Key** | Group encryption optimization where each sender has one key shared with all group members |
| **Forward secrecy** | Property ensuring past messages remain secure even if long-term keys are compromised |
| **At-least-once delivery** | Guarantee that a message will be delivered one or more times (dedup at client for exactly-once display) |
| **Store-and-forward** | Server pattern where messages are stored temporarily and forwarded to the recipient when available |

## Appendix C: References

- [15_slack](/15_slack) -- team messaging with different architecture trade-offs
- [16_chat_system](/16_chat_system) -- general chat system design
- [17_websocket_notification_system](/17_websocket_notification_system) -- WebSocket infrastructure
- [19_push_notifications](/19_push_notifications) -- push notification delivery
