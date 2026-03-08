# Feature Store -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a centralized feature store that serves two distinct workloads: low-latency online feature retrieval for real-time model inference, and point-in-time correct offline feature retrieval for training. The hard part is preventing training-serving skew -- ensuring the features a model sees during training are identical to what it receives in production.

## 2. Requirements
**Functional:** Register and discover feature definitions with metadata. Ingest features from batch (Spark) and streaming (Flink) pipelines. Serve online feature vectors with sub-5ms latency. Provide point-in-time correct feature retrieval for training datasets. Feature transformation at serving time.

**Non-functional:** Online serving p99 under 5ms at 100K QPS. Training data retrieval: process 1 TB dataset in under 1 hour. Feature freshness: batch under 1 hour, streaming under 1 minute. Training-serving skew under 0.1%. 99.95% availability for online serving.

## 3. Core Concept: Dual-Store Architecture with Point-in-Time Joins
The feature store maintains two stores: an online store (Redis) for low-latency serving, and an offline store (S3/Parquet) for historical values. Both batch and streaming pipelines write to both stores. For training, a point-in-time join ensures the model sees features as they existed at each training event's timestamp -- not current values, which would leak future information. The join finds the most recent feature row where feature_timestamp is less than or equal to the event timestamp, preventing data leakage.

## 4. High-Level Architecture
```
Batch Pipeline (Spark) ----> Offline Store (S3/Parquet) <---- Training Data Service
         |                                                    (point-in-time join)
         +---> Online Store (Redis Cluster) <---- Feature Serving Service
         |                                              ^
Stream Pipeline (Flink) ----+                           |
                                                   Model Inference

Feature Registry (PostgreSQL) -- metadata, lineage, ownership
```
- **Online Store (Redis)**: Keyed by entity ID, stores latest feature values for sub-5ms lookups.
- **Offline Store (S3/Parquet)**: Partitioned by date, stores historical feature values for training joins.
- **Feature Registry**: Catalog of feature definitions, transformations, ownership, and lineage.
- **Training Data Service**: Performs point-in-time joins against the offline store using Spark.

## 5. How a Training Dataset Gets Built
1. ML engineer provides an entity DataFrame: list of (user_id, event_timestamp) pairs from training events.
2. Training Data Service loads the entity DataFrame and requested feature groups from S3/Parquet.
3. For each (user_id, event_timestamp), it finds the most recent feature row where feature_timestamp <= event_timestamp.
4. This is implemented as an asof-join in Spark using window functions with partition and ordering.
5. Features from multiple groups are joined together into a single training row.
6. The output dataset is written to S3 for model training, with metadata recording which feature versions were used.

## 6. What Happens When Things Fail?
- **Online store (Redis) failure**: Redis Cluster with automatic failover. Feature defaults returned if Redis is entirely unavailable. Models must handle nulls gracefully.
- **Batch pipeline failure**: Retry failed Spark jobs. Online store serves stale features (within TTL) until pipeline recovers. Alert on freshness violation.
- **Streaming pipeline failure**: Kafka retains events; Flink restarts from checkpoint. Brief staleness for streaming features.

## 7. Scaling
- **10x**: Redis Cluster with 100+ shards. Materialized feature vectors for top models to eliminate multi-key lookups. Delta Lake with Z-ordering for efficient offline joins.
- **100x**: Tiered online store with hot features in Redis and warm features in SSD-backed RocksDB. Feature serving as a sidecar co-located with model serving pods.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Dual-store (Redis + S3/Parquet) | Optimizes for both latency and historical analysis, but requires maintaining two data paths and consistency between them |
| Point-in-time joins for training | Prevents data leakage and ensures training-serving consistency, but is computationally expensive for large time ranges |
| Unified transformation definitions (batch + serving) | Eliminates training-serving skew in feature logic, but requires discipline to avoid divergence and adds a testing burden |

## 9. Closing (30s)
> We designed a feature store with a dual-store architecture: Redis for sub-5ms online serving at 100K QPS, and S3/Parquet for point-in-time correct training datasets. Batch and streaming pipelines write to both stores with unified transformation definitions to prevent training-serving skew. A centralized registry provides feature discovery, lineage, and versioning across 200+ ML teams. The system scales to billions of entities through sharded online storage and partitioned offline storage.
