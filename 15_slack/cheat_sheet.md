# Slack — Cheat Sheet

## Key Numbers
- 20M DAU, 500K workspaces, 1B messages/day
- Message QPS: ~11.6K avg, ~30K peak
- Concurrent WebSocket connections: ~5M
- Message storage: ~182 TB/year (persistent)
- Search queries: 100M/day
- Target: message delivery < 200ms, search < 500ms

## Core Components
- **Gateway Servers**: WebSocket termination (~50K connections each)
- **Channel Service**: message CRUD, access control, persistence to sharded MySQL
- **Fanout Service**: Kafka consumer delivering messages to online members via Gateways
- **Search Service**: Elasticsearch-backed full-text search with workspace isolation
- **Search Indexer**: Kafka consumer indexing messages into Elasticsearch (< 5s lag)
- **Read State Service**: Redis-based unread counts and last-read markers
- **Presence Service**: online/away/DND status via Redis Pub/Sub
- **Notification Service**: desktop, mobile push, and email digest routing

## Architecture in One Sentence
A channel-centric persistent messaging platform where messages are stored in sharded MySQL, fanned out to online members via Kafka-driven WebSocket delivery, and indexed into Elasticsearch for full-text search with workspace-scoped access control.

## Top 3 Trade-offs
1. **Persistent vs ephemeral messages**: enables search and compliance but costs 182 TB/year growing storage
2. **Kafka vs Redis Pub/Sub**: Kafka for durable message fanout (ordering guarantee), Redis Pub/Sub for ephemeral events (typing, presence)
3. **Query-time vs index-time access control**: filtering at query time avoids re-indexing when membership changes but creates large filter clauses

## Scaling Story
- **10x**: archive old messages to columnar storage, shard Elasticsearch by workspace + time, regional Gateway deployment
- **100x**: custom search engine replacing Elasticsearch, custom time-series message store, dedicated infrastructure for large enterprise workspaces

## Closing Statement
Slack is a persistent, searchable messaging platform where Kafka-driven fanout delivers channel messages to online WebSocket clients and Elasticsearch indexes everything for full-text search. Unlike WhatsApp, messages are server-readable for enterprise compliance, and the architecture separates message delivery from search so each can fail independently.
