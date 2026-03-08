# Push Notifications -- Cheat Sheet

## Key Numbers
- 2B registered device tokens, 10B pushes/day, 500K/sec peak
- Notification payload ~1KB, device token ~200 bytes
- Device token storage: 400 GB in Cassandra
- Delivery log: 2 TB/day, 30-day retention (60 TB)
- APNs: 20 HTTP/2 connections x 500 streams = 10K concurrent requests
- FCM: 500 tokens per multicast batch request

## Core Components (one line each)
- **Notification Service**: Validates, persists metadata, publishes to Kafka fan-out queue
- **Fan-out Workers**: Resolve targets (user/segment/broadcast) into device tokens, split by platform
- **Device Registry (Cassandra)**: Stores 2B device tokens partitioned by app_id with user_id secondary index
- **Provider Workers**: Platform-specific workers (APNs/FCM/Web Push) with connection pooling and retry logic
- **Rate Limiter (Redis)**: Per-app, per-user, and per-provider rate limiting with token bucket
- **Delivery Tracker (Flink)**: Streaming aggregation of sent/delivered/opened events from SDK beacons
- **Token Cleanup Service**: Removes invalid tokens based on provider 410/NOT_REGISTERED responses

## Architecture in One Sentence
A three-tier fan-out pipeline resolves notification targets into device tokens, batches them into platform-specific Kafka queues, and delivers through provider workers (APNs HTTP/2, FCM multicast, Web Push VAPID) with retry logic, rate limiting, and SDK-based delivery tracking.

## Top 3 Trade-offs
1. **At-least-once + collapse key for perceived exactly-once**: True exactly-once impossible with external providers; collapse_key deduplicates on device
2. **Direct provider integration vs third-party**: Build direct for cost control at 10B/day scale despite engineering investment
3. **Fan-out on write for all targets**: Enables real-time delivery but broadcast to 100M devices takes 10-30 min due to provider rate limits

## Scaling Story
- **10x (100B/day)**: More Kafka partitions and provider workers, 10x APNs connections, streaming segment evaluation
- **100x (1T/day)**: Multi-region provider workers, sharded Kafka by app_id, dedicated provider connection pools per region, tiered log storage

## Closing Statement
A fan-out pipeline with platform-specific provider workers, collapse-key dedup, and SDK-based delivery tracking delivers 10B daily pushes across APNs/FCM/Web Push with at-least-once reliability and real-time analytics.
