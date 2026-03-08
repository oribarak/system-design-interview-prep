# Chat System — Cheat Sheet

## Key Numbers
- 100M DAU, 50M concurrent connections, 10B messages/day
- Message QPS: ~116K avg, ~300K peak
- Chat Servers: ~1,000 (50K connections each)
- Message storage: ~3 TB/day text, ~1.1 PB/year
- Target: < 200ms delivery for online users

## Core Components
- **Chat Servers**: stateful WebSocket servers (~50K connections each)
- **Session Registry**: Redis mapping user_id to Chat Server for message routing
- **Sequence Generator**: Redis INCR per conversation for monotonic message ordering
- **Message Store**: Cassandra partitioned by conversation_id with sequence-based clustering
- **Conversation Service**: group management, membership, settings (PostgreSQL)
- **Presence Service**: online/offline/away tracking via Redis + heartbeats
- **Push Service**: APNs/FCM for offline user notification
- **Sync Service**: multi-device message synchronization via last-seen sequence tracking

## Architecture in One Sentence
Stateful WebSocket Chat Servers route messages using a Redis session registry, assign per-conversation sequence numbers via atomic Redis INCR for consistent ordering, and persist to Cassandra with group fan-out optimized to O(gateway servers).

## Top 3 Trade-offs
1. **Per-conversation sequence vs timestamps**: sequence numbers give total ordering with simple gap detection, but add a Redis dependency per message
2. **Cassandra vs MySQL for messages**: Cassandra handles append-only writes and partition queries natively, but MySQL (with Vitess) offers simpler operations and is what Discord uses
3. **Push on connect vs pull on demand**: pushing all pending messages on reconnect is simpler and faster, but wastes bandwidth if user only opens one conversation

## Scaling Story
- **10x**: 10K+ Chat Servers, tier old messages to cold storage, regional deployment with local sequence counters
- **100x**: custom binary protocol, C++/Rust connection servers (500K connections each), multi-region sequencing with region prefixes

## Closing Statement
The chat system uses per-conversation Redis sequence numbers for total message ordering, Cassandra for append-only message persistence, and stateful WebSocket Chat Servers with Redis-based session routing. Group delivery is optimized to O(gateway servers) by batching, and zero message loss is guaranteed through server-side persistence before ACK with client-side retry and deduplication.
