# System Design Interview: Slack — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We are designing Slack -- a team collaboration platform with real-time messaging organized into workspaces and channels. Unlike WhatsApp's peer-to-peer model, Slack is workspace-centric: messages are persistent, searchable, and shared among team members. The key challenges are real-time message delivery across channels, full-text message search, file sharing, thread support, and managing workspace-level access control. I will cover the channel messaging architecture, real-time synchronization, search infrastructure, and how we scale for millions of concurrent workspace users."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing the core messaging platform, or also Slack Connect, Huddles, and Workflow Builder?
  - *Assume: core messaging -- channels, DMs, threads, file sharing, search. Mentions of others briefly.*
- Should we include the notification system (desktop, mobile, email digests)?
  - *Assume: yes, notifications are part of the core experience.*

**Scale**
- How many users and workspaces?
  - *Assume: 20M DAU across 500K paid workspaces. Average workspace has 200 members.*
- Message volume?
  - *Assume: 1B messages/day across all workspaces.*

**Policy / Constraints**
- Message retention policy?
  - *Assume: messages are persistent and searchable forever (paid plans). Free plan retains 90 days.*
- Is end-to-end encryption required?
  - *Assume: no E2E encryption (enterprise compliance requires server-side access for DLP/eDiscovery). TLS in transit, AES at rest.*

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| DAU | 20M |
| Messages per day | 1B |
| Message send QPS | 1B / 86400 = ~11.6K QPS avg, ~30K peak |
| Average message size | 500 bytes (text + metadata) |
| New message storage/day | 1B * 500B = 500 GB/day |
| Message storage/year | ~182 TB/year (all retained) |
| File uploads/day | 50M files, avg 2MB = 100 TB/day |
| Concurrent WebSocket connections | ~5M (25% of DAU online simultaneously) |
| Channels per workspace | avg 500 |
| Search queries per day | 100M |

Compared to WhatsApp, Slack has far fewer concurrent connections (5M vs 200M) but requires permanent storage and full-text search.

---

## 3) Requirements (3-5 min)

### Functional
- Organize messages into workspaces, channels (public/private), and DMs
- Real-time message delivery to all channel members
- Message threading (replies to a specific message)
- Full-text search across all messages and files in a workspace
- File upload and sharing within channels
- Message editing and deletion
- User mentions (@user, @channel, @here) with notifications
- Emoji reactions
- Read state tracking (unread counts, last-read markers)

### Non-functional
- **Low latency**: messages delivered to online users in < 200ms
- **Message persistence**: all messages stored permanently (paid plans)
- **Searchability**: full-text search returns results in < 500ms
- **High availability**: 99.95% uptime during business hours
- **Consistency**: messages must appear in the same order for all channel members
- **Scalability**: 20M DAU, 30K peak message QPS, 100M search queries/day

---

## 4) API Design (2-4 min)

### Send Message
```
POST /v1/channels/{channel_id}/messages
Body: { text, thread_ts?, attachments?: file_id[], mentions?: user_id[] }
Response: 201 { message_id, ts, channel_id }
```

### Get Channel Messages
```
GET /v1/channels/{channel_id}/messages?cursor=<ts>&limit=50&direction=backward
Response: 200 { messages: Message[], has_more, next_cursor }
```

### Search Messages
```
GET /v1/search?query=<text>&workspace_id=<id>&filters={channel,user,date_range}
Response: 200 { results: SearchResult[], total_count, next_cursor }
```

### Edit Message
```
PUT /v1/channels/{channel_id}/messages/{ts}
Body: { text }
Response: 200 { message: Message, edited: true }
```

### Upload File
```
POST /v1/files/upload
Body (multipart): { file, channels: channel_id[] }
Response: 200 { file_id, url, permalink }
```

### Mark Channel Read
```
POST /v1/channels/{channel_id}/mark
Body: { ts: "<last_read_timestamp>" }
Response: 200 { ok: true }
```

---

## 5) Data Model (3-5 min)

### Workspaces (PostgreSQL)
| Column | Type | Notes |
|---|---|---|
| workspace_id | BIGINT PK | |
| name | VARCHAR(64) | |
| plan | ENUM | free, pro, enterprise |
| settings | JSONB | retention, SSO config |

### Channels (PostgreSQL, sharded by workspace_id)
| Column | Type | Notes |
|---|---|---|
| channel_id | BIGINT PK | |
| workspace_id | BIGINT FK | shard key |
| name | VARCHAR(80) | |
| type | ENUM | public, private, dm, group_dm |
| topic | TEXT | |
| created_at | TIMESTAMP | |

### Messages (sharded by channel_id)
| Column | Type | Notes |
|---|---|---|
| channel_id | BIGINT | shard key, composite PK |
| ts | DECIMAL(16,6) | message timestamp, composite PK (Slack's unique message ID) |
| user_id | BIGINT | |
| text | TEXT | |
| thread_ts | DECIMAL(16,6) | parent message ts (null if top-level) |
| edited_at | TIMESTAMP | null if never edited |
| attachments | JSONB | file references |
| reactions | JSONB | emoji -> [user_ids] |

### Channel Membership (PostgreSQL, sharded by workspace_id)
| Column | Type | Notes |
|---|---|---|
| channel_id | BIGINT | composite PK |
| user_id | BIGINT | composite PK |
| last_read_ts | DECIMAL(16,6) | for unread tracking |
| notification_pref | ENUM | all, mentions, none |
| joined_at | TIMESTAMP | |

### Read State Cache (Redis)
- Key: `unread:{user_id}:{channel_id}`
- Value: count of messages with ts > last_read_ts
- Updated on new message and mark-as-read

### Search Index (Elasticsearch)
- Index: `messages_{workspace_id}`
- Fields: text, user_id, channel_id, ts, file_content (extracted text)
- Sharded by workspace_id for tenant isolation

**Storage choices**: PostgreSQL for structured data (workspaces, channels, memberships) with strong consistency. Messages in MySQL/PostgreSQL sharded by channel_id (Slack historically uses MySQL with Vitess). Elasticsearch for full-text search. Redis for real-time state (unread counts, presence, typing). S3 for file storage.

---

## 6) High-Level Architecture (5-8 min)

**Dataflow**:
1. User sends message via WebSocket to Gateway Server
2. Gateway routes to Channel Service, which persists the message
3. Channel Service publishes to a Kafka topic partitioned by channel_id
4. Real-Time Fanout Service consumes the event, looks up channel members' connected Gateway Servers, and pushes the message via WebSocket
5. Search Indexer consumes the same event and indexes into Elasticsearch
6. Notification Service determines who needs to be notified (mentions, DMs, away users)

**Components**:
- **Gateway Servers**: WebSocket termination, handle 50K connections each
- **Channel Service**: message CRUD, channel management, access control
- **Real-Time Fanout Service**: delivers messages to online channel members
- **Session Service**: maps user_id to connected Gateway Server (Redis)
- **Search Service**: Elasticsearch-backed full-text search
- **Search Indexer**: Kafka consumer that indexes messages into Elasticsearch
- **Notification Service**: determines notification routing (desktop, mobile, email)
- **File Service**: handles upload, virus scanning, preview generation
- **Presence Service**: tracks online/away/DND status
- **Read State Service**: tracks unread counts per user per channel
- **PostgreSQL/MySQL Cluster**: messages, channels, workspaces, memberships
- **Redis Cluster**: session state, presence, unread counts, typing indicators
- **Elasticsearch Cluster**: message search index
- **Kafka**: event bus for real-time fanout, search indexing, analytics
- **S3**: file storage
- **CDN**: file delivery

**One-sentence summary**: A channel-centric messaging system where messages are persisted in sharded MySQL, fanned out to online members via WebSocket through Kafka-driven fanout workers, and indexed into Elasticsearch for full-text search.

---

## 7) Deep Dive #1: Real-Time Channel Fanout (8-12 min)

When a message is posted to a channel, all online members must receive it in real-time. This is different from WhatsApp's 1:1 routing -- a channel can have thousands of members.

### Architecture

**Pub/Sub per Channel**:
1. Message posted to #general (500 members)
2. Channel Service persists message and publishes to Kafka topic `messages`, partition = hash(channel_id)
3. Fanout Consumer reads the event
4. Fanout Consumer queries channel membership: 500 members
5. Fanout Consumer queries Session Service: which of the 500 are online and on which Gateway Server?
6. Result: 200 online, spread across 15 Gateway Servers
7. Fanout Consumer sends the message to each of the 15 Gateway Servers
8. Each Gateway Server pushes to its connected clients via WebSocket

### Optimization: Channel Subscription Registry
Instead of querying membership + sessions for every message, maintain a real-time registry:
- When a user connects and has channel X open, register: `channel_subscribers:{channel_id}` -> `{user_id: gateway_server_id}`
- When a message arrives for channel X, look up the subscriber registry directly
- Registry is maintained in Redis and updated on connect/disconnect/channel-switch

### Scaling Challenges

**Large channels (10K+ members)**:
- 10K online members across ~200 Gateway Servers
- Fanout is O(num_gateway_servers), not O(num_members) -- key optimization
- Message is sent once per Gateway Server, which locally multicasts to its connected members

**#general channel with 50K members**:
- Only ~12K online at any time
- Spread across ~240 Gateway Servers
- Fanout latency: < 50ms to reach all Gateway Servers (parallel dispatch)

### Message Ordering
- Within a channel, messages must appear in the same order for all members
- Kafka partition per channel_id ensures single-producer ordering
- Server assigns the definitive timestamp (`ts`) -- clients do not determine order
- If two messages arrive simultaneously, the server serializes them on the Kafka partition

### Typing Indicators
- Lightweight, ephemeral -- not persisted
- User types -> WebSocket event to Gateway -> Fanout to channel subscribers via Redis Pub/Sub (not Kafka)
- Redis Pub/Sub is faster for ephemeral events (no persistence overhead)
- Auto-expire after 5 seconds if no continued typing

### Thread Fanout
- Thread replies are fanned out differently:
  - To channel: show "X replied to a thread" in channel timeline
  - To thread subscribers: full message delivered
  - Thread subscribers = original poster + all who replied + explicitly subscribed users

---

## 8) Deep Dive #2: Full-Text Message Search (5-8 min)

Search is a core differentiator for Slack. Users expect to find any message ever sent.

### Search Architecture

**Indexing Pipeline**:
1. New message event consumed from Kafka by Search Indexer
2. Indexer enriches the message: resolve @mentions to names, extract link previews, extract file text (PDF, doc)
3. Index into Elasticsearch with workspace-level index: `messages_{workspace_id}`
4. For large workspaces (> 1M messages), sub-index by date range: `messages_{workspace_id}_{year_quarter}`

**Query Pipeline**:
1. User submits search query with optional filters (channel, user, date range, has:file, has:link)
2. Search Service constructs Elasticsearch query with:
   - Full-text match on message text
   - Access control filter: only channels the user is a member of
   - Workspace isolation: query only the user's workspace index
3. Results ranked by relevance (BM25) with recency boost
4. Return highlighted snippets with context

### Access Control in Search
Critical: users must only see messages from channels they belong to.
- Option A: filter at query time using a `channel_id IN [user's channels]` clause
- Option B: pre-filter with a Bitset of user's accessible channel_ids
- Slack uses Option A with channel_id as a filterable field in Elasticsearch. For users in many channels (500+), this creates large filter clauses but Elasticsearch handles it efficiently with Bitset caching.

### Search Index Design
```json
{
  "channel_id": 12345,
  "workspace_id": 67890,
  "user_id": 111,
  "text": "Hey team, the deployment went well",
  "ts": 1678900000.000000,
  "thread_ts": null,
  "has_file": false,
  "has_link": false,
  "mentions": [222, 333],
  "reactions": ["thumbsup", "tada"]
}
```

### Indexing Consistency
- Messages are searchable within 2-5 seconds of being sent (near-real-time)
- Edited messages: update the Elasticsearch document in place
- Deleted messages: delete from Elasticsearch immediately
- Channel access changes (user leaves/joins): affect query-time filtering, no re-indexing needed

### File Search
- Files uploaded to Slack are processed by a text extraction pipeline
- Supported formats: PDF, DOC, PPTX, TXT, code files
- Extracted text is indexed alongside the message that shared the file
- Image files: OCR is run to extract text from screenshots/photos

---

## 9) Trade-offs and Alternatives (3-5 min)

### Persistent vs Ephemeral Messages
| Aspect | Persistent (Slack) | Ephemeral (WhatsApp) |
|---|---|---|
| Storage cost | High (182 TB/year growing) | Low (deleted after delivery) |
| Search | Full-text search possible | Not possible |
| Compliance | Supports eDiscovery, DLP | Cannot comply |
| Privacy | Server has access | Server is zero-knowledge |

Slack's enterprise customers require persistence for compliance. This is a product decision that shapes the entire architecture.

### Elasticsearch vs Solr vs Custom Search
- Elasticsearch: rich query DSL, near-real-time indexing, excellent for text search. Operational complexity at scale.
- Solr: similar capabilities, slightly harder to operate in distributed mode.
- Custom (like Slack's actual move from Elasticsearch to a custom search system at scale): better control over relevance and access control.
- Decision: Elasticsearch for the design interview, with acknowledgment that custom may be needed at extreme scale.

### Kafka vs Redis Pub/Sub for Fanout
| Aspect | Kafka | Redis Pub/Sub |
|---|---|---|
| Persistence | Yes (replay) | No (fire-and-forget) |
| Ordering | Per-partition guaranteed | No guarantee |
| Throughput | Very high | High but limited by memory |
| Use case | Message fanout (durable) | Typing indicators (ephemeral) |

Use both: Kafka for message delivery (must not lose), Redis Pub/Sub for typing/presence (loss acceptable).

### Channel-per-partition vs Workspace-per-partition in Kafka
- Channel-per-partition: strong ordering per channel, but millions of channels = too many partitions
- Workspace-per-partition: manageable partition count, but messages from different channels interleave
- Solution: hash(channel_id) % N partitions per workspace. Gives ordering within a channel while limiting partition count.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (200M DAU, 10B messages/day)
- **Message storage**: 1.8 PB/year. Partition messages by channel + time range. Archive old messages to columnar storage (Parquet on S3) with on-demand hydration.
- **Search index**: Elasticsearch cluster grows to thousands of nodes. Shard by workspace + time. Implement search-tier caching for frequently queried workspaces.
- **WebSocket connections**: 50M concurrent. 1,000+ Gateway Servers. Regional deployment for lower latency.
- **Fanout**: Kafka consumers scale horizontally. Pre-compute channel subscription sets to avoid per-message membership queries.

### At 100x
- **Custom search**: replace Elasticsearch with a purpose-built search engine optimized for channel-based access control and recency-biased ranking.
- **Message storage**: move to a custom time-series storage optimized for chat (append-only, channel-partitioned, compressed).
- **Real-time infrastructure**: custom WebSocket servers with optimized memory per connection. Multiplex multiple channels over a single connection.
- **Multi-tenancy**: physically isolate large enterprise workspaces on dedicated infrastructure.

---

## 11) Reliability and Fault Tolerance (3-5 min)

**Failure modes**:
- **Gateway Server crash**: 50K users disconnect and reconnect to another server. Session Service updates routing. Messages sent during reconnection are queued and delivered.
- **Kafka broker failure**: replication factor 3. Automatic partition leader election. Messages may be delayed by seconds.
- **Elasticsearch cluster failure**: search unavailable but messaging continues. Messages queue in Kafka for indexing when search recovers.
- **Database shard failure**: PostgreSQL streaming replication with automatic failover. Message writes fail over to replica within seconds.

**No message loss guarantee**:
1. Client sends message -> Gateway ACKs after Kafka write
2. Kafka replicates to 2+ brokers before ACK
3. Channel Service consumer persists to database
4. If consumer fails, Kafka retains the message for retry

**Graceful degradation**:
1. Full functionality (ideal)
2. Search unavailable -- messaging and notifications work
3. File uploads unavailable -- text messaging works
4. Presence/typing unavailable -- messages still delivered
5. Real-time delivery delayed -- messages retrievable via REST API polling

---

## 12) Observability and Operations (2-4 min)

**Key Metrics**:
- Message delivery latency (sender to all channel members): p50, p99
- Search query latency and result quality (click-through rate)
- WebSocket connection count and reconnection rate
- Unread count accuracy (compare calculated vs actual)
- Elasticsearch indexing lag
- Kafka consumer lag per topic

**Alerting**:
- Message delivery p99 > 500ms -- page on-call
- Search p99 > 2s -- warn
- Elasticsearch indexing lag > 30s -- warn
- Gateway Server reconnection rate spike > 5x baseline -- investigate
- Unread count drift detected -- trigger reconciliation

**Dashboards**:
- Message throughput by workspace tier (free/pro/enterprise)
- Channel size distribution and large channel performance
- Search query patterns and top queries
- File upload and processing pipeline

---

## 13) Security (1-3 min)

- **Authentication**: SSO via SAML/OIDC for enterprise workspaces. OAuth 2.0 for API integrations.
- **Authorization**: workspace admins control channel creation, guest access, file sharing policies. Channel-level access control enforced on every API call and search query.
- **Data Loss Prevention**: enterprise plans integrate DLP scanning to detect sensitive data (credit cards, SSNs) in messages.
- **eDiscovery**: enterprise compliance tools allow legal holds and message export for audits.
- **Encryption**: TLS 1.3 in transit, AES-256 at rest. Enterprise Key Management (EKM) allows customers to control their own encryption keys.
- **Audit logs**: all admin actions and message accesses logged for compliance.
- **App security**: third-party apps (bots, integrations) use scoped OAuth tokens with granular permissions.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure**:
- Messaging Infrastructure team: real-time delivery, Kafka, Gateway Servers
- Search team: Elasticsearch, indexing pipeline, relevance
- Platform team: workspace management, channels, permissions
- Enterprise team: compliance, DLP, eDiscovery, EKM
- Integrations team: bot framework, app directory, webhooks

**Deployment**:
- Canary: 1% of Gateway Servers, monitor connection stability and message delivery
- Database schema changes: online DDL via gh-ost or pt-online-schema-change
- Search index changes: re-index in background, swap alias when complete

**On-call**: separate rotations for messaging (delivery SLA), search (quality), and enterprise (compliance).

---

## 15) Common Follow-up Questions

**Q: How does Slack handle @channel notifications in a 10K-member channel?**
A: When @channel is used, the notification is generated for all members. To prevent notification storms, Slack limits @channel usage (configurable by admins), shows a confirmation dialog ("You are about to notify 10,000 people"), and batches notification delivery. Users with notifications set to "mentions only" still receive the notification since @channel is an explicit mention.

**Q: How do you implement the unread count badge?**
A: Each channel membership record has a `last_read_ts`. When a new message arrives, increment a Redis counter `unread:{user_id}:{channel_id}`. When the user views the channel, update `last_read_ts` and reset the counter. Periodically reconcile Redis counters against the database to correct drift. The workspace-level badge is the sum of all channel unread counts.

**Q: How do you handle message edits and deletions in search?**
A: Message edits publish a MessageUpdated event to Kafka. The search indexer updates the Elasticsearch document in place. Deletions publish a MessageDeleted event, and the indexer removes the document. Both operations are near-real-time (< 5 seconds).

**Q: How would you design Slack Connect (cross-workspace channels)?**
A: Shared channels have a `shared_channel_id` that maps to both workspaces. Messages are stored once but indexed in both workspace search indices. Access control checks membership in the shared channel across both workspaces. The challenge is that different workspaces may have different retention and compliance policies -- the stricter policy applies.

---

## 16) Closing Summary (30-60s)

> "To summarize, we designed Slack as a channel-centric, persistent messaging platform. Messages are stored permanently in sharded MySQL/PostgreSQL, enabling the full-text search capability powered by Elasticsearch that is central to Slack's value proposition. Real-time delivery uses WebSocket connections through Gateway Servers, with Kafka-driven fanout that delivers messages to all online channel members within 200ms. The fanout is optimized to be O(gateway servers), not O(members), by batching messages per server. Search is near-real-time with workspace-scoped Elasticsearch indices and query-time access control filtering. Unlike WhatsApp, Slack deliberately keeps messages server-readable for enterprise compliance features like DLP and eDiscovery. The system degrades gracefully -- search can go down without affecting messaging, and vice versa."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Description |
|---|---|---|
| `max_channel_members` | 10,000 | Maximum members per channel |
| `message_retention_days` | unlimited (paid) | Message retention policy |
| `search_index_lag_target` | 5s | Target time from message to searchable |
| `gateway_max_connections` | 50,000 | WebSocket connections per Gateway Server |
| `unread_reconcile_interval` | 1 hour | Frequency of unread count reconciliation |
| `typing_indicator_ttl` | 5s | Auto-expire typing indicator |
| `file_max_size_mb` | 1,000 | Max file upload size (enterprise) |
| `at_channel_confirm_threshold` | 50 | Show confirmation dialog above this member count |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Channel fanout** | Delivering a message to all online members of a channel via their Gateway Servers |
| **Message ts** | Slack's unique message identifier -- a decimal timestamp (e.g., 1678900000.000123) |
| **Thread** | A sub-conversation attached to a parent message via `thread_ts` |
| **Workspace isolation** | Ensuring data (messages, files, search results) never leak between workspaces |
| **DLP** | Data Loss Prevention -- scanning messages for sensitive content |
| **eDiscovery** | Legal process for searching and exporting messages during litigation |
| **EKM** | Enterprise Key Management -- customer-controlled encryption keys |
| **Gateway Server** | WebSocket termination server that bridges clients to backend services |

## Appendix C: References

- [14_whatsapp](/14_whatsapp) -- peer-to-peer messaging comparison
- [16_chat_system](/16_chat_system) -- general chat architecture
- [17_websocket_notification_system](/17_websocket_notification_system) -- WebSocket infrastructure
- [05_search_autocomplete](/05_search_autocomplete) -- search infrastructure
