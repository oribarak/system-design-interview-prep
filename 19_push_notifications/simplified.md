# Push Notifications -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to deliver push notifications to 2 billion devices across iOS (APNs), Android (FCM), and web (Web Push). The hard part is abstracting over multiple provider APIs with different protocols and rate limits, handling massive fan-out for broadcast campaigns to 100M+ devices, and tracking delivery across systems we do not control.

## 2. Requirements
**Functional:** Send targeted notifications to specific devices or users, segment-based and broadcast push, schedule time-zone-aware delivery, support rich notifications (images, actions, deep links), and track delivery/open analytics.

**Non-functional:** At-least-once delivery to provider gateways, targeted push p99 under 2 seconds to provider, sustain 500K pushes/sec peak, per-app rate limiting, graceful handling of provider outages.

## 3. Core Concept: Three-Tier Fan-Out Pipeline
The key insight is a three-stage pipeline: (1) resolve the target into device tokens, (2) batch tokens by platform into provider-specific queues, (3) deliver through dedicated provider workers with connection pooling and retry logic. This decouples target resolution (which can be slow for broadcast) from actual delivery, and isolates provider-specific concerns (APNs HTTP/2 vs FCM batch API) into their own workers.

## 4. High-Level Architecture
```
Producer --> [API Gateway] --> Notification Service --> [Kafka]
                                                          |
                                              Fan-out Workers
                                             /       |       \
                                    [APNs Q]   [FCM Q]   [Web Push Q]
                                       |          |            |
                                  APNs Workers  FCM Workers  Web Workers
                                       |          |            |
                                     APNs       FCM       Web Push
```
- **Notification Service**: persists metadata, publishes to fan-out queue.
- **Fan-out Workers**: resolve targets to device tokens via Device Registry (Cassandra).
- **Provider Workers**: platform-specific workers maintaining persistent connections to APNs/FCM/Web Push.
- **Device Registry (Cassandra)**: stores 2B device tokens partitioned by app_id.
- **Delivery Tracker**: aggregates sent/delivered/opened counts from SDK callbacks.

## 5. How a Push Notification Is Delivered
1. Producer calls the Send API with target (user, segment, or broadcast) and notification content.
2. Notification Service persists metadata and publishes a job to Kafka.
3. Fan-out Workers resolve the target into device tokens and batch them by platform.
4. Per-device tasks are published to platform-specific Kafka queues (APNs, FCM, Web Push).
5. Provider Workers consume batches, send to the provider API using persistent HTTP/2 (APNs) or batch HTTP (FCM) connections.
6. Provider responses are processed: success updates delivery count, 410 (gone) marks token invalid, 429 triggers backoff and retry.
7. Client SDK reports delivery and open events back to the Delivery Tracker for analytics.

## 6. What Happens When Things Fail?
- **APNs outage**: Messages queue in Kafka with TTL. When APNs recovers, delivery resumes. Stale notifications expire gracefully.
- **FCM rate limiting**: Exponential backoff with jitter. Spread load across multiple sender IDs.
- **Invalid device tokens**: Provider returns "not registered". Token Cleanup Service marks tokens invalid and removes them within 24 hours.

## 7. Scaling
- **10x**: Scale fan-out workers and Kafka partitions horizontally. 10x APNs HTTP/2 connections. Move segment evaluation from batch to streaming (Flink) for fresher segments.
- **100x**: Multi-region deployment with fan-out workers and provider connections per region. Dedicated provider connection pools per APNs/FCM region endpoint. Tiered delivery log storage (Cassandra 7 days, S3 Parquet for analytics).

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Build direct provider integration (not third-party) | Full control and no per-message cost, but significant engineering investment |
| Kafka over SQS for queuing | Higher throughput and replay capability, but more operational overhead |
| Collapse keys for dedup | Provider replaces previous notification on device, but client may miss intermediate updates |

## 9. Closing (30s)
> We designed a push notification platform handling 10B pushes/day through a three-tier fan-out pipeline: target resolution to device tokens, platform-specific batching via Kafka, and provider workers with connection pooling and retry logic. Cassandra stores 2B device tokens, collapse keys enable client-side dedup for perceived exactly-once delivery, and a streaming analytics pipeline tracks delivery and open rates. The system degrades gracefully under provider outages by queuing with TTL and prioritizing transactional over marketing notifications.
