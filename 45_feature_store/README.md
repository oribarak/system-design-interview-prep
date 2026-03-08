# System Design Interview: Feature Store for ML Training & Inference -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a feature store that provides a centralized repository for machine learning features, serving two distinct workloads: batch feature retrieval for training pipelines and low-latency online feature serving for real-time inference. The system must prevent training-serving skew, support point-in-time correct feature retrieval, handle feature transformations, and scale to serve thousands of models across hundreds of ML teams. I will cover the offline/online storage architecture, the feature transformation pipeline, and the point-in-time join mechanism that ensures consistency."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | How many features do we manage? | 50,000+ feature definitions across 500 feature groups. |
| 2 | How many entities (users, items, etc.)? | 1B user entities, 100M item entities, 10M merchant entities. |
| 3 | Online serving latency target? | p99 < 5ms for a feature vector (up to 100 features). |
| 4 | Online QPS? | 100K QPS at peak. |
| 5 | How are features computed -- batch, streaming, or both? | Both. Batch features updated hourly/daily, streaming features updated in real-time. |
| 6 | Do we need point-in-time correctness for training? | Yes, critical to avoid data leakage. |
| 7 | How many training jobs run concurrently? | ~100 daily, reading TBs of feature data. |
| 8 | Multi-team? | Yes, 200+ ML teams, features shared and discoverable. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Feature Catalog:**
- 50,000 features across 500 feature groups.
- Feature group = set of features for one entity type (e.g., user_profile_features).

**Online Store:**
- 1B user entities x 200 features avg x 8 bytes = 1.6 TB (user features).
- 100M item entities x 100 features x 8 bytes = 80 GB (item features).
- Total online store: ~2 TB.
- 100K QPS, each request fetches 50-100 features for 1-5 entities.
- Per request: ~5 entity lookups x 100 features x 8 bytes = 4 KB. Total: 100K x 4 KB = 400 MB/s.

**Offline Store:**
- Historical feature values for training (point-in-time joins).
- 1B entities x 365 days x 200 features x 8 bytes = 580 TB/year.
- Stored in Parquet on S3/GCS, partitioned by date and entity.

**Feature Computation:**
- Batch: 500 feature groups x hourly/daily = ~2,000 computation jobs/day.
- Streaming: 50 feature groups updated in real-time from Kafka events.

## 3) Requirements

### Functional
- FR1: Register, discover, and manage feature definitions with metadata.
- FR2: Ingest batch-computed features from data pipelines (Spark, Flink).
- FR3: Ingest streaming features from real-time event streams.
- FR4: Serve online feature vectors with sub-5ms latency.
- FR5: Provide point-in-time correct feature retrieval for training datasets.
- FR6: Feature transformation: apply transformations at serving time (normalization, encoding).
- FR7: Feature lineage and versioning.

### Non-Functional
- NFR1: Online serving p99 < 5ms at 100K QPS.
- NFR2: Training data retrieval: process 1 TB dataset in < 1 hour.
- NFR3: Feature freshness: batch < 1 hour, streaming < 1 minute.
- NFR4: Training-serving skew < 0.1% (features seen in training match serving).
- NFR5: 99.95% availability for online serving.

## 4) API Design

```
# Feature Registration
POST /v1/feature-groups
Body:
{
  "name": "user_spending",
  "entity": "user_id",
  "features": [
    {"name": "total_spend_30d", "type": "float64", "description": "Total spend in last 30 days"},
    {"name": "avg_order_value", "type": "float64", "description": "Average order value"},
    {"name": "num_orders_7d", "type": "int64", "description": "Order count in last 7 days"}
  ],
  "source": {"type": "batch", "schedule": "0 * * * *"},  // hourly
  "ttl": "48h",
  "owner": "fraud-team"
}

# Online Feature Retrieval
POST /v1/features/online
Body:
{
  "entity_keys": [{"user_id": "u123"}, {"user_id": "u456"}],
  "feature_refs": ["user_spending:total_spend_30d", "user_spending:avg_order_value",
                    "user_profile:age", "user_profile:country"]
}
Response:
{
  "results": [
    {"user_id": "u123", "features": {"total_spend_30d": 1250.50, "avg_order_value": 62.5, "age": 34, "country": "US"}},
    {"user_id": "u456", "features": {"total_spend_30d": 340.00, "avg_order_value": 34.0, "age": 28, "country": "UK"}}
  ],
  "metadata": {"freshness": {"user_spending": "2026-03-07T10:00:00Z"}}
}

# Offline Feature Retrieval (Training)
POST /v1/features/offline
Body:
{
  "entity_df_uri": "s3://training/entity_df.parquet",  // entity_id + event_timestamp
  "feature_refs": ["user_spending:total_spend_30d", "user_profile:age"],
  "output_uri": "s3://training/features_output.parquet"
}
Response: { "job_id": "job-789", "status": "running" }

# Feature Ingestion
POST /v1/feature-groups/{name}/ingest
Body:
{
  "source_uri": "s3://pipelines/user_spending/2026-03-07/",
  "format": "parquet"
}
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| FeatureGroup | group_id, name, entity_type, features[], source, schedule, owner | PostgreSQL |
| FeatureDefinition | feature_id, group_id, name, type, description, version | PostgreSQL |
| OnlineFeatureRow | entity_key, feature_group, feature_values (map), updated_at | Redis / DynamoDB |
| OfflineFeatureRow | entity_key, event_timestamp, feature_values, ingestion_timestamp | Parquet on S3 (partitioned) |
| IngestionJob | job_id, group_id, status, rows_processed, source_uri | PostgreSQL |
| FeatureLineage | feature_id, upstream_sources[], transformation_code | PostgreSQL |

### Storage Choices
- **Online store**: Redis Cluster (low latency, 2 TB fits in cluster). DynamoDB as alternative for managed scaling.
- **Offline store**: Parquet files on S3, partitioned by date and entity. Queryable via Spark/Trino.
- **Registry**: PostgreSQL for feature definitions, metadata, lineage.
- **Streaming buffer**: Kafka for real-time feature ingestion.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A dual-store feature system where batch and streaming pipelines write features to both an offline store (S3/Parquet) for point-in-time training retrieval and an online store (Redis) for low-latency serving, with a unified registry ensuring training-serving consistency.

### Components

1. **Feature Registry** -- metadata catalog for feature definitions, lineage, ownership.
2. **Batch Ingestion Pipeline** -- Spark jobs that compute features and write to offline + online stores.
3. **Stream Ingestion Pipeline** -- Flink/Kafka Streams that compute real-time features.
4. **Offline Store** -- S3/Parquet for historical feature values, supports point-in-time joins.
5. **Online Store** -- Redis Cluster for low-latency feature serving.
6. **Feature Serving Service** -- gRPC/REST API for online feature retrieval.
7. **Training Data Service** -- generates point-in-time correct training datasets.
8. **Transformation Service** -- applies feature transformations at serving time.
9. **Monitoring** -- feature freshness, drift detection, serving latency.

### Data Flow
1. Batch pipeline (Spark) computes features hourly, writes to offline store (S3) and materializes latest values to online store (Redis).
2. Streaming pipeline (Flink) processes events from Kafka, updates online store in real-time, appends to offline store.
3. Online serving: model calls Feature Serving Service -> lookups from Redis -> returns feature vector.
4. Training: ML engineer specifies entity DataFrame with timestamps -> Training Data Service performs point-in-time join against offline store -> outputs training dataset.

## 7) Deep Dive #1: Point-in-Time Feature Retrieval (8-12 min)

### The Problem
When building a training dataset, we need to know what features looked like at the time of each training example. Using current feature values causes data leakage -- the model sees future information during training.

**Example:** Training a fraud model on a transaction from January 15. We need the user's spending features as of January 15, not today's values.

### Point-in-Time Join Algorithm

**Input:**
- Entity DataFrame: list of (entity_id, event_timestamp) pairs from training events.
- Feature references: which feature groups to join.

**Algorithm:**
```
For each (entity_id, event_timestamp) in entity_df:
  For each feature_group:
    Find the most recent feature row WHERE
      feature_row.entity_key = entity_id
      AND feature_row.event_timestamp <= event_timestamp
    Join that feature row to the entity_df row
```

**Implementation with Spark:**
1. Load entity DataFrame and feature DataFrames (Parquet from S3).
2. For each feature group, perform an asof-join:
   - Sort both DataFrames by entity_id and timestamp.
   - For each entity, binary search for the latest feature row before the event timestamp.
3. Use window functions: `ROW_NUMBER() OVER (PARTITION BY entity_id ORDER BY feature_timestamp DESC)` with filter `feature_timestamp <= event_timestamp`.

**Optimization:**
- Partition feature data by date. For a training window of 90 days, only scan 90 partitions.
- Coalesce small partitions to reduce S3 list/read overhead.
- Cache frequently joined feature groups in a columnar format (Delta Lake / Iceberg).
- Pre-compute materialized training datasets for popular feature combinations.

### Handling Feature Freshness Windows
- Features have different update frequencies (hourly, daily, weekly).
- For an event at 10:30 AM, the hourly feature last updated at 10:00 AM is used.
- For slowly-changing features (e.g., user demographics), the last known value is carried forward.

### Preventing Training-Serving Skew
Skew occurs when the feature computation logic differs between training and serving:

1. **Unified transformation definition**: Feature transformations defined once in code (e.g., Python functions or SQL), used by both batch pipeline and serving.
2. **Transformation versioning**: Each feature has a transformation version. Training datasets record which version was used.
3. **Skew detection**: Periodically compare online-served features against batch-computed values for the same entity+timestamp. Alert if divergence > threshold.
4. **Feature logging**: Log served feature vectors at inference time. Replay these for training instead of recomputing (reduces skew to zero but costs storage).

## 8) Deep Dive #2: Online Feature Serving Performance (5-8 min)

### Redis Data Layout
```
Key: fs:{entity_type}:{entity_id}:{feature_group}
Value: {
  "total_spend_30d": 1250.50,
  "avg_order_value": 62.5,
  "num_orders_7d": 8,
  "_ts": "2026-03-07T10:00:00Z"
}
```

- Hash type in Redis for efficient partial reads.
- Entity-level partitioning for data locality.
- TTL set per feature group (e.g., 48h for spending features).

### Request Optimization
A single model inference may need features from 5 feature groups for 3 entities = 15 Redis lookups.

**Optimizations:**
1. **Pipeline/batch**: Use Redis MGET or pipeline to fetch all keys in one round trip.
2. **Feature vector pre-materialization**: For common feature combinations, pre-compute and cache the full vector as a single key.
3. **Client-side caching**: For slowly-changing features (demographics), cache locally with 5-minute TTL.
4. **Connection pooling**: Maintain persistent connections to Redis cluster.

### Serving Architecture
- gRPC service with connection multiplexing.
- Feature Serving Service is stateless -- all state in Redis.
- Horizontal scaling: add more serving instances behind load balancer.
- Feature transformation applied in the serving layer (normalize, one-hot encode) using the same transformation code as batch.

### Handling Missing Features
- **Default values**: Defined per feature in the registry. Return default if key not found.
- **Feature staleness**: If feature timestamp > TTL, return default and log a staleness alert.
- **Partial response**: Return available features even if some are missing. Model must handle nulls.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Online store | Redis (fast, memory) | DynamoDB (managed, durable) | Redis for latency, DynamoDB as fallback |
| Offline store | Data lake (S3/Parquet) | Data warehouse (BigQuery) | S3/Parquet for flexibility and cost |
| Feature transforms | Pre-computed only | Real-time transforms at serving | Both: pre-compute batch, real-time for streaming |
| Point-in-time joins | Full history scan | Partitioned + materialized views | Partitioned with Delta Lake for efficiency |
| Training-serving consistency | Shared code only | Feature logging + replay | Shared code + periodic skew detection |
| Feature freshness | Batch only (hourly) | Batch + streaming | Both: streaming for time-sensitive features |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (1M QPS, 10B entities)
- Redis Cluster with 100+ shards.
- Read replicas for online serving (separate read and write paths).
- Materialized feature vectors for top models (eliminate multi-key lookups).
- Offline store on Delta Lake with Z-ordering for efficient joins.

### 100x (10M QPS, 100B entities)
- Tiered online store: hot features in Redis, warm features in SSD-backed store (e.g., RocksDB).
- Feature serving as sidecar: co-locate feature cache with model serving pods.
- Streaming-first architecture: most features computed in real-time, batch as backfill only.
- Feature store as a service mesh: each model gets a dedicated feature cache warmed by predicted access patterns.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Online store failure**: Redis Cluster with automatic failover (Sentinel). Read from replica on primary failure. Feature defaults returned if Redis is entirely unavailable.
- **Batch pipeline failure**: Retry failed Spark jobs. Online store serves stale features (within TTL) until pipeline recovers. Alert on freshness violation.
- **Streaming pipeline failure**: Kafka retains events. Flink restarts from checkpoint. Brief staleness for streaming features.
- **Registry failure**: PostgreSQL with standby replica. Feature definitions cached in serving layer at startup.
- **Data corruption**: Checksums on Parquet files. Online store values validated against expected ranges on ingestion.
- **Multi-AZ deployment**: Redis Cluster spans AZs. S3 is inherently multi-AZ.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Feature freshness**: Time since last update per feature group (online and offline).
- **Serving latency**: p50/p99 per feature group and per entity type.
- **Feature coverage**: Percentage of requests where all requested features are available.
- **Skew metric**: Divergence between online and offline feature values (sampled).
- **Feature usage**: Which features are used by which models. Identify unused features for cleanup.
- **Ingestion throughput**: Rows/s for batch and streaming pipelines.

### Dashboards
- Feature health: freshness, coverage, staleness alerts per feature group.
- Serving performance: latency, QPS, error rate.
- Data quality: feature value distributions, drift detection.

### Alerts
- Feature freshness > 2x expected update interval.
- Online-offline skew > 1% for any feature.
- Serving latency p99 > 10ms.
- Feature coverage < 95% for any model.

## 13) Security (1-3 min)

- **Access control**: RBAC per feature group. Teams can only read/write their own features. Shared features require explicit grants.
- **PII features**: Tagged in registry. Encrypted at rest, access audited. Automatic masking in training datasets for unauthorized users.
- **Data lineage**: Track which raw data sources feed into each feature for compliance.
- **Serving authentication**: mTLS between model serving and feature store.
- **Audit log**: All feature access logged with requester identity, model, and timestamp.

## 14) Team and Operational Considerations (1-2 min)

- **Core Platform Team** (3-4 engineers): Online/offline stores, ingestion pipelines, serving API.
- **Data Engineering Team** (2-3 engineers): Batch feature pipelines, data quality, point-in-time joins.
- **Streaming Team** (2-3 engineers): Flink/Kafka streaming features.
- **SDK/DevEx Team** (2 engineers): Python SDK, feature authoring experience, registry UI.
- **ML Platform SRE** (2 engineers): Operations, capacity planning, monitoring.

## 15) Common Follow-up Questions

1. **How do you handle feature dependencies (feature B depends on feature A)?**
   - DAG-based pipeline orchestration (Airflow). Feature B's job runs after A completes. Registry tracks dependency graph.

2. **How do you handle feature versioning when a transformation changes?**
   - New version of the feature. Old version continues serving until all models migrate. Both versions can coexist in online store.

3. **How do you handle features for new entities (cold start)?**
   - Return default values for unknown entities. For new users, use population-level aggregate features until enough data accumulates.

4. **How do you ensure consistency between batch and streaming features?**
   - Lambda architecture: streaming values overwrite batch values as they arrive. Batch pipeline serves as the "source of truth" that periodically corrects any streaming drift.

5. **How do you handle feature store costs at scale?**
   - TTL-based expiration for old online values. Tiered offline storage (hot on SSD, warm on S3 Glacier). Feature usage tracking to deprecate unused features.

## 16) Closing Summary (30-60s)

"We designed a feature store with a dual-store architecture: an S3-based offline store with point-in-time join capability for training datasets, and a Redis-based online store for sub-5ms feature serving at 100K QPS. The system ingests features from both batch (Spark) and streaming (Flink) pipelines, with unified transformation definitions that prevent training-serving skew. A centralized registry provides feature discovery, lineage tracking, and versioning across 200+ ML teams. The architecture scales to billions of entities by sharding the online store and partitioning the offline store, with materialized feature vectors and tiered storage at higher scale."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| online_store_ttl | 48h | Longer = more stale data, shorter = more cache misses |
| batch_schedule | hourly | More frequent = fresher, higher compute cost |
| streaming_checkpoint_interval | 1 min | Shorter = less data loss on restart, more I/O |
| pit_join_lookback | 90 days | Training window; larger = more data to scan |
| materialization_threshold | 10 models | Pre-materialize if > N models use same feature combo |
| redis_pipeline_batch_size | 50 | Group Redis commands; larger = fewer round trips |
| skew_detection_sample_rate | 1% | Higher = more accurate skew detection, more cost |
| feature_default_strategy | zero | Options: zero, mean, null; depends on model tolerance |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| Feature group | Collection of related features for one entity type |
| Entity | The thing features describe (user, item, merchant) |
| Point-in-time join | Retrieve feature values as they were at a specific past timestamp |
| Training-serving skew | Difference between features used in training vs. serving |
| Online store | Low-latency key-value store for real-time feature serving |
| Offline store | Historical feature warehouse for training data generation |
| Feature freshness | How recently a feature was updated |
| Materialization | Pre-computing and storing feature vectors for fast retrieval |
| Feature lineage | Tracking data sources and transformations that produce a feature |
| Lambda architecture | Combining batch and streaming processing for features |

## Appendix C: References

- Feast (open-source feature store)
- Tecton (managed feature platform)
- Uber Michelangelo feature store (engineering blog)
- Spotify feature store architecture
- Hopsworks feature store
- Netflix feature management at scale
