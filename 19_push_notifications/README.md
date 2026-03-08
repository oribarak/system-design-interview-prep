# System Design Interview: Push Notifications -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a push notification platform that delivers messages to hundreds of millions of devices across iOS (APNs), Android (FCM), and web (Web Push) -- think order updates, breaking news, social alerts, and marketing campaigns. The core challenges are reliable delivery through third-party gateways with varying reliability, handling massive fan-out for broadcast campaigns, respecting user preferences and rate limits, and providing delivery analytics. I will cover the ingestion pipeline, provider gateway abstraction, fan-out strategies, and delivery tracking. Let me start with clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we building an internal platform for one app or a multi-tenant service (like Firebase/OneSignal)? *Multi-tenant platform serving multiple apps.*
- Which push channels? *iOS (APNs), Android (FCM), and Web Push (VAPID). SMS and email out of scope.*
- Do we need rich notifications (images, action buttons, deep links)? *Yes, full rich notification support.*

**Scale**
- How many registered devices? *2 billion device tokens across all tenants.*
- Daily push volume? *10 billion pushes/day. Peak: 500K pushes/sec.*
- What percentage are targeted (1-to-1) vs. broadcast (all users)? *70% targeted, 20% segmented (subset), 10% broadcast.*

**Policy / Constraints**
- Delivery latency target? *Targeted: p99 < 5 seconds. Broadcast to 100M devices: complete within 30 minutes.*
- Do we need delivery receipts? *Yes -- track sent, delivered, opened, and dismissed.*
- Rate limiting? *Yes -- per-app rate limits to prevent spam. APNs and FCM have their own rate limits we must respect.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Registered devices | Given | 2B |
| Daily pushes | Given | 10B/day |
| Average pushes/sec | 10B / 86400 | ~115K/s |
| Peak pushes/sec | ~4x average | 500K/s |
| Notification payload | avg 1KB (with rich media URL) | 1KB |
| Peak bandwidth to providers | 500K x 1KB | 500 MB/s |
| Device token storage | 2B x 200 bytes | 400 GB |
| Notification log storage/day | 10B x 200 bytes (metadata) | 2 TB/day |
| Notification log retention | 30 days | 60 TB |
| Provider connections (APNs) | HTTP/2 multiplexed, 500 streams/conn | 1,000 connections for 500K/s |
| Provider connections (FCM) | HTTP/1.1 batch API, 1000 tokens/request | 500 requests/s for 500K/s |

## 3) Requirements (3-5 min)

### Functional
- **Send targeted notifications**: Push to a specific device token or user (all their devices).
- **Segment-based push**: Send to a user segment (e.g., "users in Texas who haven't opened the app in 7 days").
- **Broadcast push**: Send to all registered devices for an app.
- **Scheduling**: Schedule pushes for future delivery (e.g., "send at 9am in the user's local timezone").
- **Rich notifications**: Support images, action buttons, deep links, and custom payloads.
- **Delivery tracking**: Track sent, delivered, opened, dismissed states with analytics dashboard.

### Non-functional
- **Reliability**: At-least-once delivery to provider gateways. No silent drops.
- **Low latency**: Targeted pushes delivered to provider within p99 < 2 seconds.
- **Throughput**: Sustain 500K pushes/sec peak.
- **Multi-tenancy**: Isolate apps; per-app rate limits and quotas.
- **Provider abstraction**: Unified API regardless of device platform.
- **Fault tolerance**: Graceful handling of provider outages (APNs down, FCM throttling).

## 4) API Design (2-4 min)

```
POST /api/v1/notifications/send
  Body: {
    app_id: "app_123",
    target: {
      type: "device" | "user" | "segment" | "broadcast",
      device_tokens: [...],        // for type=device
      user_ids: [...],             // for type=user
      segment_id: "seg_456",       // for type=segment
    },
    notification: {
      title: "Your order shipped",
      body: "Package arriving Tuesday",
      image_url: "https://...",
      deep_link: "app://orders/789",
      actions: [{label: "Track", action: "track_order"}],
      data: {order_id: "789"},     // custom payload
    },
    options: {
      priority: "high" | "normal",
      ttl_seconds: 86400,
      collapse_key: "order_update_789",
      scheduled_at: "2025-01-15T09:00:00Z",
      timezone_aware: true,
    }
  }
  Returns: {notification_id, status: "queued", estimated_recipients}

GET /api/v1/notifications/{notification_id}/status
  Returns: {
    total_targeted: 150000,
    sent: 148000,
    delivered: 142000,
    opened: 35000,
    failed: 2000,
    in_progress: false
  }

POST /api/v1/devices/register
  Body: {app_id, user_id, platform: "ios"|"android"|"web", device_token, device_info}
  Returns: {device_id}

DELETE /api/v1/devices/{device_token}
  Returns: {ok}

PUT /api/v1/users/{user_id}/preferences
  Body: {app_id, categories: {marketing: false, transactional: true, ...}}
  Returns: {ok}
```

## 5) Data Model (3-5 min)

### Device Tokens Table (Cassandra)
| Column | Type | Notes |
|--------|------|-------|
| app_id | UUID | Partition key |
| device_token | VARCHAR | Clustering key |
| user_id | UUID | For user-level targeting |
| platform | ENUM | ios, android, web |
| token_status | ENUM | active, invalid, expired |
| device_info | JSON | OS version, app version, locale, timezone |
| created_at | TIMESTAMP | |
| last_active_at | TIMESTAMP | Updated on app open |

**Secondary index:** `user_id -> [device_tokens]` (stored in a separate table or materialized view for user-level lookup).

### Notifications Table (Cassandra)
| Column | Type | Notes |
|--------|------|-------|
| notification_id | TIMEUUID | PK |
| app_id | UUID | |
| target_type | ENUM | device, user, segment, broadcast |
| title | TEXT | |
| body | TEXT | |
| payload | JSON | Full notification content |
| options | JSON | priority, TTL, collapse_key |
| status | ENUM | queued, sending, sent, failed |
| total_targeted | BIGINT | |
| sent_count | BIGINT | |
| delivered_count | BIGINT | |
| opened_count | BIGINT | |
| created_at | TIMESTAMP | |

### Delivery Log (Cassandra, TTL 30 days)
| Column | Type | Notes |
|--------|------|-------|
| notification_id | UUID | Partition key |
| device_token | VARCHAR | Clustering key |
| status | ENUM | sent, delivered, opened, failed |
| provider_response | VARCHAR | APNs/FCM response code |
| sent_at | TIMESTAMP | |
| delivered_at | TIMESTAMP | |
| opened_at | TIMESTAMP | |

### User Preferences (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| user_id | UUID | PK (composite with app_id) |
| app_id | UUID | PK |
| preferences | JSONB | Per-category opt-in/out |
| quiet_hours | JSONB | {start, end, timezone} |
| updated_at | TIMESTAMP | |

## 6) High-Level Architecture (5-8 min)

**Dataflow:** A producer calls the Send API with a notification and target. The API validates, persists the notification metadata, and publishes a job to the Notification Queue (Kafka). The Fan-out Workers resolve the target into individual device tokens (looking up segments, user devices, etc.), batch them, and publish per-device jobs to platform-specific queues (APNs queue, FCM queue, Web Push queue). Provider Workers consume from these queues and send notifications to the respective provider APIs (APNs HTTP/2, FCM HTTP, Web Push VAPID). Provider responses (success, invalid token, rate limited) are processed: invalid tokens are marked for cleanup, rate-limited requests are retried with backoff. Delivery receipts from providers are consumed and stored in the delivery log.

**Components:**
- **API Gateway**: Authentication, rate limiting, request validation.
- **Notification Service**: Persists notification metadata, publishes to fan-out queue.
- **Fan-out Workers**: Resolve targets to device tokens, create per-device delivery tasks.
- **Segment Service**: Pre-computed user segments based on attributes and behavior.
- **Device Registry**: Stores and manages device tokens and platform info.
- **Provider Workers (APNs/FCM/Web Push)**: Platform-specific workers that speak each provider's protocol.
- **Rate Limiter**: Per-app and per-provider rate limiting to prevent abuse and respect provider limits.
- **Delivery Tracker**: Aggregates delivery receipts and updates notification analytics.
- **Scheduler**: Holds time-delayed and timezone-aware notifications until their send time.
- **Token Cleanup Service**: Removes invalid/expired tokens based on provider feedback.

**One-sentence summary:** Producers submit notifications to a fan-out pipeline that resolves targets into device tokens, batches them into platform-specific queues, and delivers through APNs/FCM/Web Push provider workers with retry logic, rate limiting, and delivery tracking.

## 7) Deep Dive #1: Fan-out and Provider Gateway Management (8-12 min)

### The Fan-out Challenge

A broadcast notification to 100M devices must complete within 30 minutes. That requires sustained throughput of 100M / 1800s = ~56K deliveries/sec for a single notification, while the system simultaneously handles normal targeted traffic.

### Fan-out Architecture

**Tier 1 -- Target Resolution:**
- For `type=device`: Direct -- tokens are already provided.
- For `type=user`: Lookup `user_id -> [device_tokens]` in the Device Registry.
- For `type=segment`: Query the Segment Service, which returns a cursor over matching user_ids. The segment is pre-computed and stored as a bitmap or sorted set in Redis.
- For `type=broadcast`: Stream all device tokens for the app from Cassandra using token-range scans (no full table scan -- use Cassandra's partition token ranges to parallelize).

**Tier 2 -- Batching and Queuing:**
- Device tokens are grouped by platform: iOS tokens to APNs queue, Android to FCM queue, web to Web Push queue.
- Tokens are batched: FCM supports sending to up to 1,000 tokens per request. APNs uses HTTP/2 multiplexing (500 concurrent streams per connection).
- Each batch is a Kafka message with ~1,000 device tokens and the notification payload.

**Tier 3 -- Provider Delivery:**
- Provider Workers consume batches and call the respective APIs.
- Connection pooling: maintain persistent HTTP/2 connections to APNs and HTTP connections to FCM.
- Concurrency: each worker handles 100 concurrent provider requests.

### Provider-Specific Handling

**APNs (Apple Push Notification service):**
- Protocol: HTTP/2 with JWT or certificate authentication.
- Send individual requests per device token (but multiplexed over HTTP/2).
- Responses: 200 (success), 410 (token expired -- remove), 429 (rate limited -- back off).
- Connection limit: Apple recommends maintaining a small number of connections (10-20) with high multiplexing.

**FCM (Firebase Cloud Messaging):**
- Protocol: HTTP v1 API with OAuth2 authentication.
- Supports sending to individual tokens or topics.
- Batch endpoint: send to up to 500 tokens per multicast request.
- Responses: success, `NOT_REGISTERED` (remove token), `QUOTA_EXCEEDED` (back off).

**Web Push (VAPID):**
- Protocol: HTTP POST to the subscription endpoint with VAPID authentication.
- Each subscription has a unique endpoint URL.
- Payload encrypted with subscription's public key (RFC 8291).
- Responses: 201 (queued), 404/410 (subscription expired -- remove).

### Rate Limiting Strategy

Three levels of rate limiting:
1. **Per-app**: Prevent any single tenant from monopolizing the pipeline. Configurable limits (e.g., 100K/min for free tier, 10M/min for enterprise).
2. **Per-provider**: Respect APNs/FCM rate limits. Use token bucket algorithm. If rate limited, exponential backoff with jitter.
3. **Per-user**: Max 10 pushes/hour per user per app to prevent notification fatigue. Excess notifications are dropped or collapsed.

### Retry Strategy

```
Retry policy:
  - Provider returned 429 (rate limited): retry after Retry-After header, max 3 retries
  - Provider returned 500/503 (server error): retry with exponential backoff (1s, 4s, 16s), max 5 retries
  - Network timeout (>10s): retry once, then dead-letter
  - Provider returned 400 (bad request): do not retry, log error
  - Provider returned 410 (gone): do not retry, mark token invalid
```

Dead-lettered notifications are persisted for investigation and can be manually retried.

## 8) Deep Dive #2: Delivery Tracking and Analytics (5-8 min)

### Delivery State Machine

```
QUEUED -> SENT -> DELIVERED -> OPENED
                             -> DISMISSED
         -> FAILED (with reason)
         -> DROPPED (rate limited / user opted out)
```

### How Each State Is Tracked

**SENT**: Recorded when the provider returns a success response (APNs 200, FCM success).

**DELIVERED**:
- APNs: No native delivery receipt. We use a "silent notification" technique -- send a silent data-only push alongside the visible notification. When the app wakes to process the silent push, it reports delivery via our SDK.
- FCM: Provides delivery data via the FCM Diagnostics API (aggregate, not per-device). For per-device tracking, our SDK reports on receipt.
- Web Push: No delivery receipt. Track via service worker `push` event handler in our SDK.

**OPENED**: Our SDK detects when the user taps the notification (via `userNotificationCenter` on iOS, notification click handler on Android/web) and reports an "opened" event to our analytics endpoint.

**DISMISSED**: On Android and web, the SDK can detect when a notification is swiped away. iOS does not provide this callback.

### Analytics Pipeline

1. SDK events (`delivered`, `opened`, `dismissed`) are sent to our analytics ingestion endpoint as lightweight HTTP beacons.
2. Events are written to Kafka topic `notification_events`.
3. A Flink streaming job aggregates events per notification_id into counters (sent, delivered, opened, failed).
4. Aggregated counters are written to a time-series database (InfluxDB or TimescaleDB) for real-time dashboards.
5. Raw events are archived to S3 for ad-hoc analysis.

### Conversion Tracking

For each notification, compute:
- **Delivery rate**: delivered / sent (target > 95% for active devices)
- **Open rate**: opened / delivered (industry average ~5-15%)
- **Conversion rate**: custom events after open / opened (requires app-side event tracking)

A/B testing: Send variant A to 50% of segment, variant B to the other 50%. Compare open rates and conversion rates. The Notification Service assigns variants using consistent hashing on user_id.

## 9) Trade-offs and Alternatives (3-5 min)

### Direct Provider Integration vs. Third-Party Service (OneSignal, Airship)
- **Direct**: Full control, no per-message cost, but significant engineering investment.
- **Third-party**: Faster to launch, built-in analytics, but vendor lock-in and per-message fees that become expensive at 10B pushes/day.
- We build direct because at our scale, the cost savings justify the engineering investment.

### Push vs. Pull
- Push (our approach): Lower latency, better UX, but requires token management and provider integration.
- Pull (polling): Simpler, but drains battery and adds latency. Unacceptable for time-sensitive notifications.

### Kafka vs. SQS for Queuing
- Kafka: Higher throughput, replay capability, better for analytics pipeline. But requires operational overhead.
- SQS: Managed, simpler, but 256KB message limit and lower throughput per queue.
- We use Kafka for the primary pipeline (throughput) and SQS-like dead-letter semantics for failed deliveries.

### CAP / PACELC
- **Device Registry (Cassandra)**: AP -- eventual consistency acceptable; a slightly stale token list means one retry.
- **Notification metadata**: CP -- we need accurate delivery counts. Use PostgreSQL for per-notification aggregates.
- **PACELC**: Favor latency over consistency on the hot delivery path; reconcile counts asynchronously.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (100B pushes/day, 5M/sec peak)
- **Fan-out workers**: Scale horizontally; add more Kafka partitions and consumer groups.
- **Provider connections**: 10x APNs connections (200 connections x 500 streams), proportional FCM scaling.
- **Device Registry**: Cassandra scales linearly; add nodes for 4TB token storage.
- **Segment pre-computation**: Move from batch to streaming segment evaluation (Flink) for fresher segments.

### 100x (1T pushes/day, 50M/sec peak)
- **Multi-region deployment**: Fan-out workers and provider connections in each major region to reduce cross-region latency.
- **Sharded queues**: Partition Kafka topics by app_id to isolate tenant workloads.
- **Provider connection pooling**: Dedicated connection pools per provider region (APNs has US and EU endpoints).
- **Tiered storage**: Hot delivery logs in Cassandra (7 days), cold in S3 (Parquet format for analytics).
- **Cost**: At 1T pushes/day x $0.000001/push (internal cost), operating cost is ~$1M/day. Provider-side costs are zero (APNs/FCM are free), but bandwidth and compute dominate.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| APNs outage | iOS notifications delayed | Queue messages in Kafka; retry when APNs recovers. TTL prevents stale delivery. |
| FCM rate limiting | Android delivery slowed | Exponential backoff with jitter; spread load across multiple sender IDs. |
| Kafka broker failure | Pipeline stall | Replication factor 3; consumer failover to new leader. |
| Fan-out worker crash | Partial fan-out | Kafka consumer group rebalance; uncommitted offsets are reprocessed. |
| Device Registry unavailable | Cannot resolve user -> tokens | Cache hot user-token mappings in local LRU cache; serve from cache during outage. |

### Exactly-once Delivery Semantics

True exactly-once is impossible because APNs/FCM are external systems. Our guarantee:
- **At-least-once to provider**: Kafka consumer commits offset only after provider acknowledges.
- **Provider dedup**: Use `collapse_key` / `apns-collapse-id` so the provider replaces duplicate notifications on the device.
- **Client-side dedup**: Our SDK deduplicates by notification_id before displaying.

### Graceful Degradation
1. Under provider pressure: reduce batch sizes, increase intervals between requests.
2. Under extreme load: shed marketing/low-priority pushes; always deliver transactional notifications.
3. Provider completely down: queue with TTL; alert ops team; notifications expire gracefully if not sent within TTL.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Throughput**: Pushes sent/sec per provider (alert if drops > 20% from baseline).
- **Provider error rate**: per provider (alert if > 1% errors sustained 5 min).
- **End-to-end latency**: API receipt to provider ACK (p50, p95, p99).
- **Invalid token rate**: percentage of tokens returning "not registered" (alert if > 5% -- indicates stale token cleanup issue).
- **Kafka consumer lag**: per partition per consumer group (alert if lag > 100K messages).
- **Queue depth**: outstanding notifications per provider queue.

### Dashboards
- Provider health: per-provider success/error/retry rates over time.
- Fan-out progress: for active broadcast campaigns, show completion percentage.
- Token health: active vs. invalid vs. expired tokens over time per app.

### Runbooks
- **APNs returning 429s**: Reduce connection count; check if APNs certificate is near expiry.
- **Spike in invalid tokens**: Likely an app update changed token format; investigate and bulk-clean.
- **High consumer lag**: Scale up provider workers; check for provider-side issues.

## 13) Security (1-3 min)

- **Authentication**: API keys with per-app scoping. Keys rotated quarterly. OAuth2 for dashboard access.
- **Token security**: Device tokens stored encrypted at rest (AES-256). Tokens are opaque identifiers -- no PII.
- **Payload encryption**: APNs and FCM encrypt payloads in transit. For sensitive content, the notification body says "You have a new message" and the app fetches actual content after opening (notification service extension on iOS).
- **Abuse prevention**: Per-app rate limiting. Abuse detection for apps sending excessive pushes (auto-throttle + alert).
- **Provider credentials**: APNs certificates and FCM service account keys stored in a secrets manager (Vault). Rotated on schedule; never stored in code.
- **PII handling**: Notification content may contain PII. Delivery logs are retained for 30 days only. GDPR: delete all data for a user on request.

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Platform team (API, fan-out pipeline, rate limiter) -- 4-5 engineers. Provider integration team (APNs, FCM, Web Push workers) -- 2-3 engineers. Analytics team (delivery tracking, dashboards) -- 2 engineers.
- **Deployment**: Provider workers deployed independently per provider for isolation. Rolling deploys with canary (5% traffic for 15 min).
- **On-call**: Single on-call rotation. Provider outages are the most common incident -- usually resolved by waiting and letting retries handle it.
- **Rollout**: Start with targeted pushes only. Add segment-based after validating fan-out at scale. Broadcast last (highest risk of overloading providers).

## 15) Common Follow-up Questions

**Q: How do you handle timezone-aware scheduled notifications?**
A: The Scheduler stores the notification with `scheduled_at` in UTC and `timezone_aware=true`. At processing time, it looks up each target user's timezone (from device_info) and computes the local send time. Users are bucketed by timezone and released in hourly batches (e.g., "send at 9am" means 9am ET, then 9am CT an hour later, etc.). This spreads load naturally.

**Q: How do you handle token migration (e.g., user gets a new phone)?**
A: When a user installs the app on a new device, the SDK registers the new token and optionally sends the old token for deregistration. If the old token is not explicitly deregistered, it will eventually return `NotRegistered` from the provider, and our Token Cleanup Service removes it within 24 hours.

**Q: How do you prevent notification spam from malicious tenants?**
A: Three layers: (1) per-app rate limits enforced at the API gateway, (2) per-user rate limits (max 10 pushes/hour per user) enforced in the fan-out worker, (3) anomaly detection that flags apps with sudden 10x spikes in volume for manual review.

**Q: Can the system handle a "flash sale" scenario where 100M pushes need to go out in 1 minute?**
A: Not in 1 minute -- provider rate limits are the bottleneck, not our pipeline. APNs and FCM throttle at their discretion. Realistically, 100M pushes take 10-30 minutes. We can prioritize this campaign by temporarily reducing capacity for lower-priority traffic. We communicate realistic delivery timelines to the tenant.

## 16) Closing Summary (30-60s)

> "We designed a push notification platform handling 10 billion pushes per day across iOS, Android, and web. The architecture uses a three-tier fan-out pipeline: target resolution (users/segments to device tokens), platform-specific batching (Kafka queues per provider), and provider workers with connection pooling and retry logic. Key design decisions include Cassandra for the 2-billion-token device registry partitioned by app_id, per-provider rate limiting with exponential backoff to respect APNs/FCM limits, collapse keys for client-side dedup achieving user-perceived exactly-once delivery, and a streaming analytics pipeline for real-time delivery/open rate tracking. The system degrades gracefully under provider outages by queuing with TTL and prioritizing transactional over marketing notifications."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `apns_connections` | 20 | 5-100 | HTTP/2 connections to APNs per worker |
| `apns_streams_per_conn` | 500 | 100-1000 | Multiplexed streams per APNs connection |
| `fcm_batch_size` | 500 | 1-1000 | Tokens per FCM multicast request |
| `per_app_rate_limit` | 100K/min | 1K-10M | App-level send rate |
| `per_user_rate_limit` | 10/hour | 1-100 | Per-user notification cap |
| `retry_max_attempts` | 5 | 1-10 | Provider retry count |
| `retry_base_delay_ms` | 1000 | 100-5000 | Base delay for exponential backoff |
| `notification_ttl_sec` | 86400 | 60-604800 | Max time to attempt delivery |
| `delivery_log_ttl_days` | 30 | 7-90 | Delivery log retention |
| `fan_out_parallelism` | 100 | 10-1000 | Concurrent fan-out tasks per worker |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **APNs** | Apple Push Notification service -- Apple's push delivery infrastructure |
| **FCM** | Firebase Cloud Messaging -- Google's push delivery for Android and web |
| **VAPID** | Voluntary Application Server Identification -- auth mechanism for Web Push |
| **Device token** | Provider-issued identifier for a specific app installation on a device |
| **Collapse key** | Identifier that tells the provider to replace previous notifications with the same key |
| **Fan-out** | Expanding a single notification into per-device delivery tasks |
| **Silent push** | Data-only notification that wakes the app without user-visible alert |
| **TTL** | Time-to-live -- how long a provider will attempt to deliver an undelivered notification |
| **Dead letter** | Queue for messages that failed all retry attempts |
| **Notification service extension** | iOS mechanism to modify notification content before display |

## Appendix C: References

- Apple APNs documentation: Sending Push Notifications Using Command-Line Tools
- Firebase Cloud Messaging HTTP v1 API reference
- RFC 8030 -- Generic Event Delivery Using HTTP Push
- RFC 8291 -- Message Encryption for Web Push
- OneSignal Architecture Blog: "Delivering 10 Billion Push Notifications a Day"
- Airship Engineering: "Lessons Learned Scaling Push to Billions of Devices"
