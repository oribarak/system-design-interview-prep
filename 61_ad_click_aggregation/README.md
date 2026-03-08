# System Design Interview: Ad Click Aggregation System -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We are designing an ad click aggregation system that processes billions of click events per day, aggregates them in near-real-time, and provides advertisers with accurate click counts, spend tracking, and conversion metrics. The system must handle high throughput with exactly-once counting semantics to ensure billing accuracy, support real-time dashboards with minute-level granularity, and detect click fraud. I will focus on the streaming aggregation pipeline, the exactly-once processing guarantees, and the time-windowed aggregation strategy."

## 1) Clarifying Questions (2-5 min)

1. **Event types**: Clicks only, or also impressions and conversions? -- Primarily clicks; impressions and conversions are similar pipelines.
2. **Scale**: How many click events per day? -- 10 billion clicks/day.
3. **Aggregation granularity**: What time windows? -- 1-minute granularity for real-time dashboards; rolled up to hourly and daily for reporting.
4. **Aggregation dimensions**: What do we aggregate by? -- ad_id, campaign_id, advertiser_id, country, device_type.
5. **Latency**: How fresh must aggregated data be? -- Within 1-2 minutes of the click event.
6. **Accuracy**: How important is exact counting? -- Critical for billing. No over-counting or under-counting.
7. **Fraud detection**: Is click fraud detection in scope? -- Yes, as a supporting feature.
8. **Query patterns**: Who queries the data and how? -- Advertisers view dashboards; internal billing system queries for spend calculation.

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Daily click events | 10 billion |
| Peak click events/sec | 10B / 86400 x 3 (peak) ~ 350K events/sec |
| Average click event size | 200 bytes (ad_id, user_id, timestamp, IP, device, etc.) |
| Daily raw data | 10B x 200 bytes = 2 TB/day |
| Monthly raw data | 60 TB/month |
| Unique ads | 5 million |
| Unique campaigns | 500K |
| Unique advertisers | 100K |
| Aggregated records per minute | 5M ads x 10 dimensions ~ 50M records (but sparse; realistic ~5M) |
| Aggregated data per day | 5M records/min x 1440 min x 50 bytes ~ 360 GB |

## 3) Requirements (3-5 min)

### Functional Requirements
- **Click Ingestion**: Receive and durably store click events from ad servers.
- **Real-Time Aggregation**: Aggregate clicks by ad_id, campaign_id, and other dimensions in 1-minute tumbling windows.
- **Dashboard Queries**: Support queries like "clicks for ad X in the last hour, broken down by minute and country."
- **Billing Aggregation**: Provide exact click counts per ad for cost calculation (CPC billing).
- **Fraud Filtering**: Detect and filter fraudulent clicks before aggregation.
- **Historical Queries**: Support queries over days/weeks/months of historical data.

### Non-Functional Requirements
- **Throughput**: Process 350K events/sec at peak.
- **Latency**: Aggregated data available within 2 minutes of the click.
- **Exactly-Once Semantics**: No double-counting or lost events for billing accuracy.
- **Durability**: No data loss. Raw events stored for audit and reprocessing.
- **Availability**: 99.99% for click ingestion (cannot drop revenue-generating events).
- **Scalability**: Handle 10x traffic spikes during major ad campaigns.

## 4) API Design (2-4 min)

### Click Ingestion (from ad servers)
```
POST /clicks
Body: {
  click_id: uuid,
  ad_id: string,
  campaign_id: string,
  advertiser_id: string,
  user_id: string (hashed),
  timestamp: iso8601,
  ip_address: string,
  device_type: string,
  country: string,
  referrer_url: string
}
Response: { status: "accepted" }
```

### Query Aggregated Data
```
GET /aggregations?ad_id=<id>&start=<ts>&end=<ts>&granularity=<1min|1hour|1day>&dimensions=<country,device>
Response: {
  ad_id: string,
  time_series: [
    { timestamp, click_count, unique_users, spend },
    ...
  ],
  breakdown: { country: { US: 1234, UK: 567, ... } }
}
```

### Campaign Summary
```
GET /campaigns/{campaign_id}/summary?date=<date>
Response: { campaign_id, total_clicks, total_spend, click_through_rate, fraud_filtered }
```

## 5) Data Model (3-5 min)

### Raw Click Event (Kafka + Cold Storage)
| Field | Type | Notes |
|---|---|---|
| click_id | uuid | Dedup key (idempotency) |
| ad_id | string | Foreign key to ad metadata |
| campaign_id | string | |
| advertiser_id | string | |
| user_id | string | Hashed for privacy |
| timestamp | timestamp | Event time (from ad server) |
| ip_address | string | For fraud detection |
| device_type | string | mobile, desktop, tablet |
| country | string | Geo from IP |
| is_fraud | bool | Set by fraud filter |

### Aggregated Click Data (OLAP Store)
| Field | Type | Notes |
|---|---|---|
| ad_id | string | Aggregation key |
| window_start | timestamp | Start of 1-min window |
| dimension_key | string | e.g., "country=US,device=mobile" |
| click_count | int64 | Sum of clicks |
| unique_users | HyperLogLog | Approximate unique count |
| total_spend | decimal | click_count x CPC rate |

### Storage Choices
- **Raw events**: Kafka (7-day retention for reprocessing) + S3/GCS (permanent cold storage in Parquet format).
- **Aggregated data**: Time-series OLAP store (Apache Druid, ClickHouse, or Apache Pinot) for fast analytical queries.
- **Fraud detection state**: Redis for sliding-window counters per IP and user.
- **Ad metadata**: PostgreSQL (CPC rates, campaign budgets, advertiser info).

## 6) High-Level Architecture (5-8 min)

**One-sentence summary**: Ad servers publish click events to Kafka, a streaming aggregation pipeline (Flink) deduplicates events, filters fraud, and computes 1-minute tumbling window aggregations that are written to an OLAP store for dashboard queries and billing -- with raw events archived to cold storage for audit and reprocessing.

### Components

1. **Click Receivers** -- Stateless HTTP endpoints that accept click events from ad servers, assign click_id if missing, and produce to Kafka.
2. **Kafka Cluster** -- Durable message bus. Topics: raw-clicks, filtered-clicks, aggregated-clicks.
3. **Stream Processor (Flink)** -- Core aggregation engine. Stages: dedup, fraud filter, aggregation.
4. **Deduplication** -- Exactly-once via click_id dedup with a sliding window state store.
5. **Fraud Filter** -- Real-time click fraud detection (IP rate limiting, bot detection, click pattern analysis).
6. **Aggregation Engine** -- Tumbling window aggregation (1-minute windows) by ad_id and dimension combinations.
7. **OLAP Store** -- Time-series analytical database for serving dashboard queries.
8. **Cold Storage** -- S3/GCS in Parquet format for raw event archive.
9. **Query Service** -- API layer for advertisers and billing systems to query aggregated data.
10. **Reconciliation Job** -- Periodic batch job that recomputes aggregations from raw events and compares with streaming results.

## 7) Deep Dive #1: Streaming Aggregation with Exactly-Once Semantics (8-12 min)

### Why Exactly-Once Matters

In ad click aggregation, each click translates to money (CPC billing). Double-counting overcharges advertisers. Under-counting loses revenue. The system must provide **exactly-once processing** end-to-end.

### Architecture: Flink with Kafka Transactions

```
Kafka (raw-clicks) -> Flink Pipeline -> Kafka (aggregated-clicks) -> OLAP Store
```

**Step 1: Deduplication**
- Each click has a unique click_id (UUID generated by the ad server or click receiver).
- Flink maintains a keyed state store (RocksDB-backed) mapping click_id -> seen, with a TTL of 1 hour.
- If a click_id has been seen before, drop it. This handles at-least-once delivery from Kafka.
- State is checkpointed to durable storage (S3) every 30 seconds.

**Step 2: Fraud Filtering**
- After dedup, apply fraud rules:
  - IP rate limit: > 10 clicks from the same IP on the same ad in 1 minute -> flag as fraud.
  - User rate limit: > 5 clicks from the same user on the same ad in 1 hour -> flag.
  - Bot detection: known bot user-agent strings, data center IP ranges.
- Fraudulent clicks are tagged (is_fraud=true) and excluded from aggregation but still stored for audit.

**Step 3: Tumbling Window Aggregation**
- Group by (ad_id, dimension_key) within 1-minute tumbling windows aligned to wall clock.
- For each group, compute:
  - click_count: COUNT(*)
  - unique_users: HyperLogLog sketch
  - total_spend: click_count x CPC rate (looked up from ad metadata)
- When the window closes (1 minute + allowed lateness of 30 seconds), emit the aggregated result.

**Step 4: Exactly-Once Sink**
- Flink uses **Kafka transactions** to write aggregated results atomically with checkpoint commits.
- Two-phase commit: Flink pre-commits to Kafka on checkpoint, then commits when the checkpoint is confirmed durable.
- This ensures that even if Flink restarts from a checkpoint, no aggregated records are duplicated or lost in Kafka.

**Step 5: OLAP Ingestion**
- A separate consumer reads from the aggregated-clicks Kafka topic and writes to the OLAP store.
- OLAP store supports upsert semantics: if the same (ad_id, window_start, dimension_key) is written twice (due to Flink recovery), the second write overwrites the first. Idempotent.

### Handling Late Data

- Clicks may arrive out of order due to network delays.
- Allowed lateness: 30 seconds. Late events within this window update the aggregation.
- Events arriving > 30 seconds late: written to a side output ("late events" Kafka topic). A periodic batch job reprocesses these and updates aggregations.
- Very late events (> 1 hour): handled by the daily reconciliation batch job.

### Watermark Strategy

- Flink watermarks track event-time progress: watermark = max_event_time - 30 seconds.
- When the watermark passes a window boundary, the window fires.
- Partitioned watermarks per Kafka partition to handle uneven partition lag.

## 8) Deep Dive #2: Click Fraud Detection (5-8 min)

### Types of Click Fraud

| Type | Description | Detection Method |
|---|---|---|
| Bot clicks | Automated scripts clicking ads | User-agent analysis, IP reputation, click pattern |
| Click farms | Human workers paid to click ads | Geographic anomalies, rapid sequential clicks |
| Competitor fraud | Competitors clicking to exhaust ad budget | IP clustering, repeated clicks from same source |
| Publisher fraud | Publishers inflating click counts | Abnormal CTR ratios, traffic source analysis |

### Real-Time Detection Pipeline

1. **IP Rate Limiting**: Sliding window counter in Flink state. Key: (IP, ad_id). If > threshold in window, flag.
2. **User Rate Limiting**: Similar sliding window on (user_id, ad_id).
3. **Click Timing Analysis**: If clicks from the same source arrive at suspiciously regular intervals (e.g., exactly every 2 seconds), flag as bot.
4. **IP Reputation**: Lookup against a known-bad IP database (data center IPs, Tor exit nodes, known proxy services). This database is loaded into Flink as a broadcast stream.
5. **Device Fingerprint**: Anomalies like missing JavaScript execution, suspiciously consistent device fingerprints across different users.

### ML-Based Fraud Detection (Batch)

- A batch ML model trained on labeled fraud data runs daily.
- Features: click timing distribution, geographic diversity, conversion rate (fraudulent clicks rarely convert), CTR anomalies.
- Scores all clicks from the past 24 hours. Flags additional fraud that real-time rules missed.
- Adjusts billing retroactively: refunds advertisers for newly detected fraud.

### Fraud Metrics

- Fraud rate tracked per ad network, publisher, geography.
- Alert if fraud rate exceeds historical baseline by > 2 standard deviations.
- Dashboard for advertisers showing filtered vs. valid clicks.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Stream processor | Apache Flink | Spark Structured Streaming | Flink has true event-time processing, lower latency, native exactly-once with Kafka |
| Dedup mechanism | Flink keyed state (click_id) | External Redis dedup | Flink state is co-located with processing; avoids network round-trip |
| OLAP store | Apache Druid / ClickHouse | PostgreSQL with TimescaleDB | Purpose-built OLAP handles high-dimensional aggregation queries much faster |
| Exactly-once | Flink + Kafka transactions | Application-level dedup only | End-to-end exactly-once prevents sink-side duplicates |
| Late data handling | Allowed lateness + side output | Ignore late data | Billing accuracy requires handling late arrivals |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (3.5M events/sec)
- Scale Kafka partitions: 10x partitions on raw-clicks topic. Flink parallelism matches Kafka partition count.
- Scale Flink cluster: more task managers. State backend moves to remote RocksDB with incremental checkpointing.
- OLAP store: add more nodes. Pre-aggregate at 5-minute and 1-hour granularities to reduce query-time work.
- Click receivers: auto-scale behind a load balancer.

### 100x (35M events/sec)
- Multi-region Kafka clusters with local processing in each region. Cross-region aggregation for global reports via batch merge.
- Tiered storage for Kafka: hot data in SSDs, older data in S3.
- Pre-compute top advertiser dashboards. Cache query results.
- Custom serialization (Protobuf/FlatBuffers) to reduce per-event processing cost.
- Approximate counting (HyperLogLog, Count-Min Sketch) for non-billing metrics to reduce state size.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Kafka durability**: Replication factor 3 with min.insync.replicas=2. No acknowledged event is lost.
- **Flink checkpointing**: Incremental checkpoints to S3 every 30 seconds. On failure, Flink restarts from the last checkpoint. Exactly-once maintained via Kafka consumer offset rollback + transaction abort.
- **OLAP store failure**: OLAP stores are replicated. If write fails, aggregated records remain in Kafka and are retried.
- **Reconciliation**: Daily batch job recomputes aggregations from raw events in cold storage (S3 Parquet). Compares with streaming aggregations. Discrepancies are flagged and corrected. This is the safety net for any streaming bugs.
- **Click receiver overload**: Kafka acts as a buffer. If receivers are slow, events queue in Kafka. Add more Flink parallelism to drain the backlog.

## 12) Observability and Operations (2-4 min)

- **Pipeline metrics**: Flink throughput (events/sec), processing latency, checkpoint duration, checkpoint failure rate.
- **Kafka consumer lag**: Monitor lag on all consumer groups. Alert if lag exceeds 2 minutes (freshness SLA).
- **Dedup metrics**: Duplicate rate (% of events deduplicated). Spike could indicate ad server retry storm.
- **Fraud metrics**: Fraud detection rate, false positive rate, fraud rate by ad network.
- **Reconciliation results**: Daily delta between streaming and batch aggregations. Non-zero delta triggers investigation.
- **Alerting**: Page on Kafka consumer lag > 5 minutes, Flink checkpoint failures > 3 consecutive, fraud rate spike > 2 sigma.

## 13) Security (1-3 min)

- **Data privacy**: User IDs are hashed before storage. IP addresses retained only for fraud detection (7-day retention).
- **Access control**: Advertisers can only query their own campaigns. Role-based access control on the query API.
- **Audit trail**: All raw click events stored in immutable cold storage for billing audits.
- **Encryption**: TLS for all data in transit. AES-256 for data at rest in cold storage.
- **Click injection prevention**: Click receivers validate requests via signed tokens from the ad server. Reject unsigned or expired clicks.

## 14) Team and Operational Considerations (1-2 min)

- **Streaming platform team**: Owns Flink jobs, Kafka infrastructure, exactly-once guarantees. Focused on throughput and reliability.
- **Data engineering team**: Owns cold storage, reconciliation jobs, data quality. Focused on accuracy.
- **Fraud team**: Owns fraud detection rules and ML models. Focused on detection rate and false positives.
- **Product/analytics team**: Owns query service and advertiser dashboards. Focused on query latency and UX.
- **On-call**: Shared rotation between streaming platform and data engineering for pipeline issues.

## 15) Common Follow-up Questions

1. **How do you handle time zone differences in reporting?** -- All events stored in UTC. Query service converts to advertiser's time zone at display time. Aggregation windows are always UTC-aligned.
2. **How do you handle budget exhaustion?** -- Separate budget tracking service decrements remaining budget on each valid click. When budget is exhausted, ad serving stops showing the ad. Not part of the aggregation pipeline.
3. **How do you ensure reconciliation accuracy?** -- Reconciliation reprocesses raw events from cold storage using the same logic as streaming. MapReduce/Spark job. Outputs compared at the (ad_id, window_start) level.
4. **What if Flink state grows too large?** -- Click_id dedup state has a 1-hour TTL. Old entries expire. For aggregation state, windows fire and clear after the allowed lateness period. Total state size is bounded.
5. **How do you handle schema evolution in click events?** -- Use Avro/Protobuf with a schema registry. Backward-compatible schema changes. Flink deserializers handle schema evolution gracefully.

## 16) Closing Summary (30-60s)

"We designed an ad click aggregation system that ingests 350K events/sec via Kafka, processes them through a Flink pipeline with exactly-once semantics -- deduplicating by click_id, filtering fraud in real time, and computing 1-minute tumbling window aggregations -- writing results to an OLAP store for sub-second dashboard queries. Raw events are archived to cold storage for daily reconciliation, which serves as the accuracy safety net. The system scales horizontally via Kafka partitions and Flink parallelism, and handles late data through allowed lateness windows with a side-output reprocessing path."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Guidance |
|---|---|---|
| Tumbling window size | 1 minute | Smaller = fresher data, more overhead and records |
| Allowed lateness | 30 seconds | Longer = catches more late data, delays window finalization |
| Checkpoint interval | 30 seconds | Shorter = less data to replay on failure, more I/O |
| Dedup state TTL | 1 hour | Must exceed max expected event delay |
| Kafka partition count (raw-clicks) | 256 | Match Flink parallelism; more = more throughput |
| IP rate limit threshold | 10 clicks/min/ad | Lower = more aggressive fraud filtering, more false positives |
| Reconciliation frequency | Daily | More frequent = faster discrepancy detection, more compute |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| Tumbling Window | Fixed-size, non-overlapping time window for aggregation |
| Exactly-Once Semantics | Guarantee that each event is processed exactly once, even on failure |
| Watermark | Mechanism tracking event-time progress in stream processing |
| Allowed Lateness | Grace period after window closes for accepting late events |
| HyperLogLog | Probabilistic data structure for approximate distinct counting |
| Two-Phase Commit | Protocol ensuring atomic writes across distributed systems |
| CPC (Cost Per Click) | Billing model where advertisers pay per ad click |
| Reconciliation | Process of comparing streaming results with batch recomputation |
| Side Output | Secondary output stream in Flink for special cases (e.g., late data) |

## Appendix C: References

- Carbone et al. "Apache Flink: Stream and Batch Processing in a Single Engine" (IEEE Data Eng. Bulletin, 2015)
- Kafka documentation: Exactly-Once Semantics and Transactional Messaging
- Akidau et al. "The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost" (VLDB 2015)
- Google Ads documentation: Click measurement and invalid click detection
- Apache Druid documentation: Real-time ingestion and rollup
