# Google Analytics — Cheat Sheet

## Key Numbers
- Peak ingestion: 2M events/sec, ~1 GB/s bandwidth
- Daily events: ~86 billion; daily raw storage: ~8.5 TB compressed (5:1)
- 90-day raw retention: ~765 TB
- Tracked properties: 5 million websites/apps
- Dashboard queries: 50K/sec, < 2 second response
- Real-time data freshness: < 5 minutes; batch: < 4 hours

## Core Components (one line each)
- **JavaScript Tracker**: client-side SDK collecting pageviews, events, and user properties via beacons
- **Edge Collection (CDN)**: validates events, resolves geo from IP, publishes to Kafka
- **Kafka**: event bus decoupling collection from processing, partitioned by property
- **Flink (stream)**: real-time sessionization and rolling aggregation for live dashboard
- **Spark (batch)**: full sessionization with late events, dimensional aggregation, HLL computation
- **ClickHouse (OLAP)**: columnar store for pre-aggregated reports, sub-second queries
- **Raw Archive (S3 + Parquet)**: long-term event storage for ad-hoc queries and reprocessing

## Architecture in One Sentence
Lambda architecture: edge-collected events flow through Kafka into parallel Flink (real-time) and Spark (batch) pipelines for sessionization and aggregation, with ClickHouse serving sub-second dashboard queries on pre-aggregated data.

## Top 3 Trade-offs
1. Lambda (two pipelines, accurate batch + fast stream) vs. Kappa (single stream, simpler but less accurate)
2. Pre-aggregation (fast queries, limited dimensions) vs. raw scans (flexible, slow for large volumes)
3. Cookie-based tracking (reliable) vs. cookieless (privacy-friendly, less accurate)

## Scaling Story
- **10x**: CDN absorbs collection, scale Kafka/Flink partitions, shard ClickHouse by property
- **100x**: sample high-volume properties (10%), tiered pipelines, approximate data structures everywhere (HLL, Count-Min Sketch)

## Closing Statement
An event analytics platform using edge collection, Kafka-backed dual pipelines (Flink + Spark), sessionization with HyperLogLog user counting, and ClickHouse pre-aggregated rollups for sub-second dashboard queries.
