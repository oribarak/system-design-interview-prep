# Prompt Logging & Observability System -- Cheat Sheet

## Key Numbers
- 500M prompt-response pairs/day, ~17K entries/s peak
- ~1.85 TB/day raw data, ~165 TB in 90-day hot storage (ClickHouse)
- Real-time metrics delay: < 30s from LLM response to dashboard
- Full-text search: < 2s over 90-day window
- Retention: 90 days hot (ClickHouse), 365 days warm (S3), 7 years cold (Glacier)
- PII detection: ~1.5ms per entry, ~25 CPU cores at peak

## Core Components
Log Emitter (async ring buffer), Kafka (durable ingestion bus), Stream Processor (Flink: PII masking + enrichment), ClickHouse Cluster (hot storage + materialized views), S3 Archiver (warm/cold tiers), Search Service, Metrics API, Grafana Dashboards, Alert Manager, Feedback Service (PostgreSQL), Export Service

## Architecture in One Sentence
LLM serving emits logs asynchronously to Kafka, a stream processor masks PII and enriches entries, ClickHouse stores hot data with materialized views powering real-time dashboards, and tiered archival to S3/Glacier handles long-term retention and compliance.

## Top 3 Trade-offs
1. **Async vs. sync logging**: Async ensures zero inference latency impact but accepts potential data loss if the ring buffer overflows during extended Kafka outages.
2. **ClickHouse vs. Elasticsearch**: ClickHouse provides faster aggregation and better compression for analytics-heavy workloads; Elasticsearch offers richer full-text search but at higher cost.
3. **PII masking at ingestion vs. encryption**: Masking is simpler and ensures raw PII never exists in storage, but is irreversible; encryption preserves the option to decrypt but adds key management complexity.

## Scaling Story
- 1x: Single ClickHouse cluster, Kafka, Flink pipeline. Handles 500M entries/day.
- 10x: Sharded ClickHouse by tenant, 100+ Kafka partitions, LZ4 compression.
- 100x: Edge extraction of metrics only, full logs to S3, approximate query processing (HyperLogLog, t-digest), adaptive sampling, federated regional clusters.

## Closing Statement
"The system captures 500M entries/day with zero inference latency impact via async Kafka ingestion, masks PII in the stream processor, stores 90 days hot in ClickHouse with materialized-view-powered real-time dashboards, and archives to S3/Glacier for 7-year compliance retention."
