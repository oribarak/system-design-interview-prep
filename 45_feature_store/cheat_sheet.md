# Feature Store -- Cheat Sheet

## Key Numbers
- 50,000 features across 500 feature groups, 1B+ entities
- Online serving: 100K QPS, p99 < 5ms, ~2 TB in Redis Cluster
- Offline store: ~580 TB/year historical features in S3/Parquet
- Batch ingestion: 2,000 jobs/day; streaming: 50 feature groups real-time
- Training-serving skew target: < 0.1%

## Core Components
Feature Registry (PostgreSQL), Batch Pipeline (Spark), Stream Pipeline (Flink/Kafka), Online Store (Redis Cluster), Offline Store (S3/Parquet/Delta Lake), Feature Serving API (gRPC), Point-in-Time Join Engine, Transform Service, Freshness Monitor, Skew Detector

## Architecture in One Sentence
Batch and streaming pipelines write features to a Redis online store for sub-5ms serving and an S3 offline store for point-in-time training retrieval, with a unified registry ensuring consistent transformation logic across both paths.

## Top 3 Trade-offs
1. **Redis vs. DynamoDB online store**: Redis gives < 2ms lookups but requires memory capacity planning; DynamoDB is fully managed but adds 5-10ms latency.
2. **Feature logging vs. recomputation for training**: Logging served features eliminates training-serving skew entirely but costs ~2x storage; recomputation is cheaper but risks skew.
3. **Pre-materialized vs. on-demand point-in-time joins**: Pre-materialization speeds up repeated training runs but costs storage and freshness; on-demand is flexible but slow for large datasets.

## Scaling Story
- 1x: Redis Cluster + S3 Parquet, Spark point-in-time joins.
- 10x: Read replicas, materialized feature vectors, Delta Lake with Z-ordering.
- 100x: Tiered online store (Redis hot, RocksDB warm), feature cache sidecars, streaming-first with batch backfill.

## Closing Statement
"The dual-store architecture serves features at sub-5ms online and generates point-in-time correct training datasets, with unified transformations preventing training-serving skew across 200+ ML teams."
