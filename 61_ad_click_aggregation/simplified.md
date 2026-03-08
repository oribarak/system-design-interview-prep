# Ad Click Aggregation System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a system that processes 10 billion ad click events per day, aggregates them in near-real-time with minute-level granularity, and provides advertisers with accurate click counts and spend tracking. The hard part is guaranteeing exactly-once counting since each click translates directly to money (CPC billing) -- double-counting overcharges advertisers, under-counting loses revenue.

## 2. Requirements
**Functional:** Ingest click events from ad servers durably. Aggregate clicks by ad_id, campaign, country, and device in 1-minute tumbling windows. Provide real-time dashboards for advertisers. Exact click counts for billing (CPC). Real-time fraud filtering. Historical query support.

**Non-functional:** 350K events/sec at peak. Aggregated data available within 2 minutes of the click. Exactly-once semantics for billing accuracy. No data loss (raw events archived). 99.99% ingestion availability. Handle 10x traffic spikes.

## 3. Core Concept: Flink Streaming with Exactly-Once via Kafka Transactions
Apache Flink consumes click events from Kafka, deduplicates by click_id using keyed state (RocksDB-backed with 1-hour TTL), filters fraud in real-time, and computes 1-minute tumbling window aggregations. The key to exactly-once: Flink uses Kafka transactions with two-phase commit -- it pre-commits aggregated results to Kafka on checkpoint, then commits when the checkpoint is confirmed durable. If Flink restarts, it rolls back to the last checkpoint and replays, producing no duplicates. A daily batch reconciliation job recomputes from raw events in cold storage as a safety net.

## 4. High-Level Architecture
```
Ad Servers --> [Click Receivers] --> Kafka (raw-clicks)
                                       |
                              [Flink Pipeline]
                              Dedup --> Fraud Filter --> Aggregation (1-min windows)
                                       |
                              Kafka (aggregated-clicks) --> [OLAP Store] (Druid/ClickHouse)
                                       |
                              [S3 Cold Storage] (Parquet, permanent archive)
                                       |
                              [Query Service] --> Advertiser Dashboards
                                       |
                              [Reconciliation Job] -- daily batch recompute
```
- **Flink Pipeline**: Core engine with dedup (click_id keyed state), fraud filter (IP/user rate limits), and tumbling window aggregation.
- **OLAP Store**: Druid or ClickHouse for fast analytical queries on aggregated data. Supports upsert semantics for idempotent writes.
- **Cold Storage**: S3 in Parquet format for raw event archive and reconciliation reprocessing.
- **Reconciliation Job**: Daily batch job that recomputes aggregations from raw events and compares with streaming results.

## 5. How Click Aggregation Works
1. Ad server sends click event to Click Receiver (stateless HTTP endpoint).
2. Click Receiver assigns click_id if missing, produces to Kafka (replication factor 3, min.insync.replicas=2).
3. Flink consumer deduplicates: check click_id against keyed state store (1-hour TTL). Drop duplicates.
4. Fraud filter applies IP rate limiting (more than 10 clicks/min/ad from same IP) and bot detection.
5. Valid clicks enter 1-minute tumbling window aggregation grouped by (ad_id, dimension_key).
6. When window closes (plus 30 seconds allowed lateness), aggregated record is emitted.
7. Flink writes to Kafka using two-phase commit tied to checkpoints for exactly-once delivery.

## 6. What Happens When Things Fail?
- **Flink crash**: Restarts from last checkpoint (every 30 seconds). Kafka consumer offsets roll back. Two-phase commit ensures no duplicate writes. Brief delay but no data loss.
- **Late-arriving events (over 30 seconds)**: Written to a side output "late events" topic. A periodic batch job reprocesses and updates aggregations.
- **Streaming vs batch disagree**: Daily reconciliation compares streaming results with batch recomputation from raw events in S3. Discrepancies are flagged and corrected.

## 7. Scaling
- **10x**: Scale Kafka partitions (10x) with matching Flink parallelism. Remote RocksDB state backend with incremental checkpointing. Pre-aggregate at 5-minute and 1-hour granularities in OLAP store.
- **100x**: Multi-region Kafka clusters with local processing per region. Cross-region batch merge for global reports. Approximate counting (HyperLogLog) for non-billing metrics. Custom serialization (Protobuf) to reduce per-event cost.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Flink vs Spark Structured Streaming | Flink has true event-time processing, lower latency, and native exactly-once with Kafka |
| Flink keyed state for dedup vs external Redis | Co-located state avoids network round-trip; simpler to checkpoint atomically |
| Allowed lateness + reconciliation | Handles late data gracefully; reconciliation is the safety net for any streaming bugs |

## 9. Closing (30s)
> A Flink pipeline ingests 350K events/sec from Kafka, deduplicates by click_id, filters fraud in real-time, and computes 1-minute tumbling window aggregations with exactly-once semantics via Kafka transactions. Raw events are archived to S3 for daily reconciliation -- the safety net that catches any streaming discrepancies. The key insight is end-to-end exactly-once: Flink checkpoints plus Kafka two-phase commit plus OLAP upsert idempotency guarantee that each click is counted exactly once for billing.
