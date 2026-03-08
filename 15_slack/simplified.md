# Slack -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design Slack -- a team collaboration platform with real-time messaging organized into workspaces and channels. Unlike WhatsApp, messages are permanent, searchable, and server-readable (for enterprise compliance). The challenge is real-time delivery to all channel members (channels can have 10K+ users), full-text search across all messages ever sent, and workspace-level access control ensuring data never leaks between tenants.

## 2. Requirements
**Functional:** Organize messages into workspaces, channels (public/private), and DMs. Real-time delivery to all channel members. Full-text search across messages and files. Message threading, editing, and deletion. Mentions with notifications. Emoji reactions. Unread count tracking.

**Non-functional:** Messages delivered to online users in under 200ms, all messages stored permanently and searchable (paid plans), search results in under 500ms, 99.95% availability, 20M DAU, 30K peak message QPS, messages appear in the same order for all channel members.

## 3. Core Concept: Channel Fan-out Optimized to O(Gateway Servers)
The key insight is how channel message delivery is optimized. When a message is posted to a channel with 10K members, the system does NOT send 10K individual messages. Instead, it groups online members by their Gateway Server (the WebSocket server they are connected to). If 2,500 members are online across 50 Gateway Servers, the fan-out sends just 50 messages -- one per Gateway Server -- which locally multicasts to its connected users. This reduces fan-out from O(members) to O(gateway_servers). Combined with a channel subscription registry in Redis, the system avoids querying membership on every message.

## 4. High-Level Architecture

```
Client --> [ Gateway Server (WebSocket) ] --> [ Channel Service ] --> [ Sharded MySQL ]
                                                      |
                                                      v
                                                  [ Kafka ]
                                                      |
                                    +-----------------+------------------+
                                    v                                    v
                         [ Fanout Service ]                    [ Search Indexer ]
                                    |                                    |
                                    v                                    v
                        [ Gateway Servers ]                    [ Elasticsearch ]
```

- **Gateway Servers**: WebSocket termination, 50K connections each. Bridge clients to backend services.
- **Channel Service**: Message CRUD, channel management, access control. Persists to sharded MySQL.
- **Fanout Service**: Kafka consumer that delivers messages to online channel members via their Gateway Servers.
- **Search Indexer**: Kafka consumer that indexes every message into Elasticsearch for full-text search.
- **Elasticsearch**: Workspace-scoped search indices with query-time access control filtering.

## 5. How a Channel Message Is Delivered and Searched
1. User sends a message via WebSocket to their Gateway Server.
2. Gateway routes to Channel Service, which persists the message to sharded MySQL and publishes to Kafka.
3. Fanout Service consumes the event, looks up the channel subscription registry in Redis (which members are online and on which Gateway Server).
4. Fanout Service sends one message per Gateway Server. Each Gateway Server multicasts to its connected channel members via WebSocket.
5. Meanwhile, Search Indexer consumes the same Kafka event and indexes the message into Elasticsearch (searchable within 2-5 seconds).
6. Notification Service determines who needs push/email notifications (mentioned users, DMs, users who are away).

## 6. What Happens When Things Fail?
- **Elasticsearch goes down**: Search is unavailable but messaging continues uninterrupted. Messages queue in Kafka for indexing when search recovers.
- **Gateway Server crashes**: 50K users disconnect and reconnect to another server. Session Service updates routing. Messages sent during reconnection are queued and delivered.
- **Database shard fails**: PostgreSQL/MySQL streaming replication with automatic failover. Message writes fail over to the replica within seconds.

## 7. Scaling
- **10x (200M DAU, 10B messages/day)**: Message storage grows to 1.8PB/year -- archive old messages to columnar storage (Parquet on S3). Elasticsearch cluster grows to thousands of nodes. 50M concurrent WebSocket connections need 1,000+ Gateway Servers with regional deployment.
- **100x**: Replace Elasticsearch with a purpose-built search engine optimized for channel-based access control. Build custom time-series storage for chat messages. Physically isolate large enterprise workspaces on dedicated infrastructure.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Persistent, server-readable messages vs E2E encryption | Persistence enables full-text search and enterprise compliance (DLP, eDiscovery), which are core to Slack's value. But it means the server can read all messages, unlike WhatsApp. |
| Kafka for message delivery vs Redis Pub/Sub | Kafka provides durable, ordered delivery for messages (must not lose). Redis Pub/Sub is used for ephemeral events like typing indicators (loss is acceptable). Using both gives the right guarantees for each use case. |
| Fan-out O(gateway servers) vs O(members) | Batching by Gateway Server reduces network calls dramatically for large channels. A 10K-member channel with 2,500 online members across 50 servers needs only 50 fan-out calls instead of 2,500. |

## 9. Closing (30s)
> "Slack is a channel-centric, persistent messaging platform. Messages are stored permanently in sharded MySQL, enabling the full-text search via Elasticsearch that is central to Slack's value. Real-time delivery uses WebSocket connections through Gateway Servers with Kafka-driven fan-out optimized to O(gateway servers) instead of O(members). Search is workspace-scoped with query-time access control. The system degrades gracefully -- search can go down without affecting messaging, and vice versa."
