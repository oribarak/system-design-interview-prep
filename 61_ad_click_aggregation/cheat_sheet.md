# Ad Click Aggregation System -- Cheat Sheet

## Key Numbers
- 10B clicks/day, 350K peak events/sec
- 2 TB raw data/day, ~360 GB aggregated/day
- 5M unique ads, 100K advertisers
- Aggregation latency: < 2 minutes from click to dashboard
- 1-minute tumbling windows, 30-second allowed lateness

## Core Components
- Click Receivers: stateless HTTP endpoints producing to Kafka
- Kafka: durable message bus (raw-clicks, aggregated-clicks, fraud-clicks, late-events topics)
- Flink Pipeline: dedup (click_id keyed state) -> fraud filter (IP/user rate limits) -> tumbling window aggregation
- OLAP Store (Druid/ClickHouse): serves sub-second analytical queries for dashboards
- Cold Storage (S3 Parquet): raw event archive for audit and reconciliation
- Reconciliation Job: daily Spark job recomputes aggregations from raw events, compares with streaming output

## Architecture in One Sentence
Ad servers publish clicks to Kafka; a Flink pipeline deduplicates, filters fraud, and computes 1-minute tumbling window aggregations with exactly-once semantics via Kafka transactions; results flow to an OLAP store for real-time dashboards and billing queries, with daily batch reconciliation as an accuracy safety net.

## Top 3 Trade-offs
1. Flink keyed state dedup vs. external Redis dedup: co-located state avoids network round-trips but increases Flink state size.
2. Tumbling windows vs. sliding windows: tumbling is simpler and more efficient; sliding provides smoother aggregations but doubles state.
3. Exactly-once via Kafka transactions vs. at-least-once with idempotent sink: exactly-once is more complex but essential for billing accuracy.

## Scaling Story
10x: scale Kafka partitions and Flink parallelism, incremental checkpointing, pre-aggregate at coarser granularities.
100x: multi-region processing with batch cross-region merge, tiered Kafka storage, approximate counting for non-billing metrics.

## Closing Statement
A Flink streaming pipeline with exactly-once Kafka transactions, click_id deduplication, real-time fraud filtering, and 1-minute tumbling window aggregation delivers accurate ad click counts to an OLAP dashboard within 2 minutes at 350K events/sec, backed by daily batch reconciliation for billing integrity.
