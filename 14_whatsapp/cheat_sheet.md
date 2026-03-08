# WhatsApp — Cheat Sheet

## Key Numbers
- 2B users, 500M DAU, 100B messages/day
- Message QPS: ~1.16M avg, ~3M peak
- Concurrent WebSocket connections: ~200M
- Chat Server capacity: ~50K connections each = ~4,000 servers
- Offline queue: ~2 TB for pending messages
- Message delivery target: < 200ms for online users

## Core Components
- **Chat Servers**: stateful WebSocket servers (~50K connections each)
- **Session Service**: Redis mapping user_id to Chat Server for message routing
- **Message Router**: routes encrypted messages between Chat Servers
- **Offline Queue**: Redis sorted set storing messages for disconnected users (30-day TTL)
- **Push Notification Service**: APNs/FCM for offline user notification
- **Key Distribution Service**: manages E2E encryption public keys and pre-keys
- **Cassandra**: message persistence (store-and-forward, deleted after delivery)
- **PostgreSQL**: user accounts, group membership

## Architecture in One Sentence
Stateful WebSocket Chat Servers route E2E encrypted messages via a Session Service, with an offline queue and push notifications ensuring zero-loss delivery to disconnected users.

## Top 3 Trade-offs
1. **Store-and-forward vs permanent storage**: deleting messages after delivery preserves privacy but limits multi-device sync
2. **WebSocket vs long polling**: persistent connections cost ~4,000 servers but deliver < 200ms latency
3. **E2E encryption vs content moderation**: zero-knowledge server prevents abuse scanning; relies on client-side reporting and behavioral signals

## Scaling Story
- **10x**: 40K+ Chat Servers, overflow offline queue to Cassandra, shard Session Service regionally
- **100x**: custom binary protocol replacing WebSocket, purpose-built high-density connection servers (Erlang/BEAM), edge-deployed Chat Servers

## Closing Statement
WhatsApp is a stateful WebSocket messaging system with E2E encryption via the Signal Protocol and zero message loss through server persistence, client ACK/retry, and idempotent deduplication. Messages are stored only until delivered, with offline queues and push notifications bridging disconnected users.
