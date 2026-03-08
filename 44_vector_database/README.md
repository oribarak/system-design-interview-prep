# System Design Interview: Vector Database for Embedding Search -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We need to design a vector database that stores billions of high-dimensional embedding vectors and supports fast approximate nearest neighbor (ANN) search with metadata filtering. The system must handle real-time inserts, scale horizontally, maintain low-latency queries under heavy concurrent load, and support multiple distance metrics. I will cover the index structures (HNSW and IVF-PQ), the distributed storage and query engine, and the hybrid filtering architecture that combines vector similarity with metadata predicates."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | How many vectors do we store? | 5 billion vectors at steady state. |
| 2 | What dimensionality? | 768 to 2048 dimensions, most commonly 1024. |
| 3 | What distance metrics? | Cosine similarity, L2 (Euclidean), inner product. |
| 4 | What query throughput? | 10K QPS with sub-10ms p99 latency. |
| 5 | Do we need real-time inserts or batch-only? | Both: real-time inserts + bulk imports. |
| 6 | Do we need metadata filtering with vector search? | Yes, filter by attributes (category, timestamp, tenant_id). |
| 7 | What recall target? | > 95% recall@10 (compared to exact KNN). |
| 8 | Multi-tenant isolation? | Yes, 10,000 tenants with data isolation. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Storage:**
- 5B vectors x 1024 dimensions x 4 bytes (FP32) = 20 TB raw vectors.
- With FP16: 10 TB. With PQ compression (64 bytes/vector): 320 GB.
- Metadata per vector: ~100 bytes. 5B x 100 = 500 GB.
- HNSW graph overhead: ~200 bytes/vector (edges). 5B x 200 = 1 TB.
- Total: ~12 TB (FP16 vectors + graph + metadata) or ~2 TB with PQ.

**Traffic:**
- 10K QPS, each query searches top-100 candidates, returns top-10.
- Inserts: 10M vectors/day = ~115 inserts/s average.
- Bulk import: up to 100M vectors in one batch job.

**Compute:**
- HNSW search (1024-dim, 5B vectors, recall 95%): ~2-5ms per query on a single shard.
- With 100 shards: parallel scatter-gather, merge top-K from each shard.
- Memory per shard (50M vectors, FP16): ~100 GB vectors + 10 GB graph = ~110 GB per shard.
- 100 shards x 110 GB = 11 TB total memory (or use disk-backed with NVMe).

**Network:**
- Query: 1024 x 4 bytes = 4 KB query vector + metadata filters.
- Response: 10 results x (4 KB vector + 100 bytes metadata) = ~41 KB.
- 10K QPS x 45 KB = ~450 MB/s (manageable).

## 3) Requirements

### Functional
- FR1: Insert, update, and delete individual vectors with metadata.
- FR2: Bulk import millions of vectors efficiently.
- FR3: ANN search with configurable top-K and recall/speed trade-off.
- FR4: Support cosine, L2, and inner product distance metrics.
- FR5: Combined metadata filtering + vector search (pre-filter and post-filter).
- FR6: Multi-collection support with independent schemas and indexes.

### Non-Functional
- NFR1: Query latency < 10ms p99 at 10K QPS.
- NFR2: Recall@10 > 95%.
- NFR3: Support 5B vectors with horizontal scaling.
- NFR4: Real-time insert visibility < 1 second.
- NFR5: 99.95% availability with no data loss.
- NFR6: Multi-tenant data isolation.

## 4) API Design

```
# Collection Management
POST /v1/collections
Body:
{
  "name": "products",
  "dimension": 1024,
  "metric": "cosine",
  "schema": {
    "category": "string",
    "price": "float",
    "created_at": "datetime"
  },
  "num_partitions": 100,
  "index_type": "HNSW"
}

# Insert Vectors
POST /v1/collections/{name}/vectors
Body:
{
  "vectors": [
    {"id": "v1", "values": [0.1, 0.2, ...], "metadata": {"category": "electronics", "price": 299.99}},
    ...
  ]
}

# Search
POST /v1/collections/{name}/search
Body:
{
  "vector": [0.15, 0.22, ...],
  "top_k": 10,
  "filter": {"category": {"$eq": "electronics"}, "price": {"$lt": 500}},
  "params": {"ef": 128}       // recall/speed trade-off
}
Response:
{
  "results": [
    {"id": "v1", "score": 0.95, "metadata": {"category": "electronics", "price": 299.99}},
    ...
  ]
}

# Delete
DELETE /v1/collections/{name}/vectors/{vector_id}

# Bulk Import
POST /v1/collections/{name}/import
Body: { "source_uri": "s3://data/vectors.parquet", "batch_size": 100000 }
```

## 5) Data Model

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| Collection | collection_id, name, dimension, metric, schema, shard_count | etcd (metadata) |
| Shard | shard_id, collection_id, node_id, vector_count, status | etcd |
| Vector | vector_id, shard_id, values (float[]), metadata (JSON) | Custom storage engine |
| HNSW Index | shard_id, graph (adjacency lists), entry_point | Memory-mapped file |
| IVF Index | shard_id, centroids, inverted_lists | Memory-mapped file |
| Metadata Index | shard_id, field_name, B-tree/bitmap index | RocksDB |
| WAL Entry | sequence_num, operation, vector_id, values, metadata | Append-only log |

### Storage Engine Architecture
- **Write path**: WAL (append-only log on NVMe) -> MemTable (in-memory buffer) -> Flush to segment files.
- **Read path**: Search MemTable + immutable segments. Segments periodically merged (LSM-tree style).
- **Vector storage**: Column-oriented -- vectors stored contiguously for SIMD-friendly scanning.
- **Metadata**: RocksDB-backed inverted indexes per field.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A distributed vector database where collections are sharded across nodes, each shard maintains an HNSW index in memory with metadata B-tree indexes, queries are scatter-gathered across shards and merged, and writes go through a WAL with periodic index rebuilds.

### Components

1. **Query Coordinator** -- parses queries, routes to shards, merges results.
2. **Shard Nodes** -- store vectors, maintain HNSW indexes, execute local search.
3. **Write Coordinator** -- routes inserts to correct shard, manages WAL replication.
4. **Index Builder** -- background process to rebuild/optimize HNSW indexes.
5. **Metadata Store** -- etcd for cluster topology, collection configs.
6. **Bulk Import Service** -- parallel ingestion from object storage.
7. **Compaction Service** -- merges segments, garbage-collects deleted vectors.
8. **Replication Manager** -- synchronizes replicas for durability.

### Data Flow (Query)
1. Client sends search request to Query Coordinator.
2. Coordinator identifies relevant shards (all, or filtered by partition key).
3. Each shard searches its HNSW index, applies metadata post-filter or pre-filter.
4. Each shard returns local top-K results.
5. Coordinator merges results across shards, re-ranks, returns global top-K.

## 7) Deep Dive #1: Index Structures and Search Algorithm (8-12 min)

### HNSW (Hierarchical Navigable Small World)

**Structure:**
- Multi-layer graph. Top layers are sparse (long-range connections), bottom layer is dense (local connections).
- Each vector is a node. Edges connect to M nearest neighbors at each layer.
- Insertion: start at top layer, greedily descend to find neighbors at each layer.

**Search Algorithm:**
1. Start at entry point (top layer).
2. Greedily traverse to nearest neighbor at current layer.
3. Descend to next layer, expand search with ef (exploration factor) candidates.
4. At bottom layer, return top-K from ef candidates.

**Key Parameters:**
- M (max edges per node): 16-64. Higher = better recall, more memory.
- ef_construction: 200-500. Higher = better index quality, slower build.
- ef_search: 64-512. Higher = better recall, slower query. Tunable per query.

**Memory:**
- Per vector: 1024 dims x 2 bytes (FP16) = 2 KB + M edges x 4 bytes x layers = ~200 bytes.
- Total per vector: ~2.2 KB. For 50M vectors per shard: ~110 GB.

### IVF-PQ (for memory-constrained scenarios)

**IVF (Inverted File Index):**
- Cluster vectors into N centroids (e.g., N = 4096).
- At query time, probe only the closest nprobe centroids (e.g., 32).
- Reduces search space by 100x.

**PQ (Product Quantization):**
- Compress 1024-dim vector into M subspaces (e.g., M = 64), each quantized to 8 bits.
- Compressed vector: 64 bytes instead of 2048 bytes (FP16). 32x compression.
- Approximate distance computed via lookup tables.

**When to use IVF-PQ vs. HNSW:**
- HNSW: best recall, highest memory usage, best for latency-critical workloads.
- IVF-PQ: lower memory (32x compression), good recall (> 90%), suitable for very large datasets that do not fit in RAM.
- Hybrid: IVF-PQ for coarse search + re-rank with full vectors for top candidates.

### SIMD Acceleration
- Distance computations (dot product, L2) are the inner loop.
- AVX-512 on x86: process 16 floats in one instruction. 8-16x speedup.
- GPU acceleration for batch queries: cuBLAS for matrix-based distance computation.

### Incremental Index Updates
- New vectors added to a MemTable (flat index, brute-force search).
- When MemTable exceeds threshold (e.g., 100K vectors), flush to a new HNSW segment.
- Background merge: combine small HNSW segments into larger ones.
- Deletes: mark as tombstone, filtered at query time, garbage-collected during compaction.

## 8) Deep Dive #2: Hybrid Metadata Filtering (5-8 min)

### The Filtering Challenge
Users want: "Find top-10 most similar vectors WHERE category = 'electronics' AND price < 500."

Two approaches:

**Pre-filtering:**
- Apply metadata filter first, get candidate set.
- Search ANN only within candidates.
- Problem: if filtered set is small, HNSW graph connectivity is broken, recall degrades.
- Solution: maintain per-partition HNSW indexes for high-cardinality filters (e.g., per-tenant index).

**Post-filtering:**
- Search HNSW for top-K (with K oversampled, e.g., K x 10).
- Filter results by metadata.
- Problem: if filter is very selective (< 1% of data matches), most results are discarded, need huge oversample.

**Our hybrid approach:**
1. Estimate filter selectivity (from metadata index statistics).
2. If selectivity > 10%: post-filter with 5x oversample.
3. If selectivity 1-10%: pre-filter with bitmap-guided HNSW search.
4. If selectivity < 1%: brute-force scan within the filtered set (small enough to be fast).

### Bitmap-Guided HNSW Search
- Maintain bitmap indexes for categorical metadata fields.
- During HNSW traversal, skip nodes not matching the filter bitmap.
- Modified greedy search: only consider neighbors that pass the bitmap check.
- Recall may degrade; compensate by increasing ef_search for filtered queries.

### Partitioned Indexes
For multi-tenant workloads:
- Partition data by tenant_id.
- Each partition has its own HNSW index.
- Query routes to the tenant's partition directly.
- Avoids the filtering problem entirely for the tenant dimension.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Primary index | HNSW (memory) | IVF-PQ (disk) | HNSW for hot data, IVF-PQ for cold/archive |
| Vector precision | FP32 (accurate) | FP16 (efficient) | FP16 -- negligible quality loss |
| Filtering | Post-filter only | Hybrid pre/post | Hybrid with selectivity estimation |
| Sharding | Hash-based | Partition-based (by tenant) | Partition for multi-tenant, hash within partitions |
| Replication | Sync (strong consistency) | Async (eventual) | Async with read-your-writes guarantee |
| Storage engine | Custom LSM | RocksDB-based | Custom -- optimized for vector workloads |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (50B vectors, 100K QPS)
- 1,000 shards across 200+ nodes.
- Tiered storage: hot shards (HNSW in memory) and warm shards (IVF-PQ on NVMe).
- Query routing optimization: skip shards based on cluster-level statistics (partition pruning).
- Read replicas for query-heavy workloads.

### 100x (500B vectors, 1M QPS)
- GPU-accelerated search: FAISS on GPU for batch query processing.
- Disaggregated compute and storage: search nodes load index segments on demand from shared storage.
- Hierarchical search: coarse global index to identify candidate shards, then fine-grained search within.
- Columnar vector compression with learned quantization (OPQ, residual quantization).
- CDN-like caching for popular query patterns.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Shard failure**: Each shard replicated 3x (1 primary + 2 replicas). Primary failure triggers leader election among replicas.
- **WAL durability**: WAL fsynced to NVMe before acknowledging writes. Replicated to followers.
- **Index corruption**: Rebuild from WAL + segment files. Checksums on all persisted data.
- **Split brain**: Raft consensus for shard group leadership. Fencing tokens for stale leaders.
- **Node failure**: Controller detects via heartbeat (10s timeout). Replicas promoted. New replicas created on healthy nodes.
- **Data migration**: When rebalancing shards, copy segments to new node, replay WAL, then swap.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Query latency** (p50, p95, p99) per collection.
- **Recall@K**: Measured periodically against exact KNN on sample queries.
- **QPS** per shard, per node.
- **Memory utilization**: HNSW graph size, MemTable size, OS page cache usage.
- **Disk I/O**: Read/write throughput per node (NVMe saturation).
- **Compaction lag**: Pending segment merges.
- **Insert throughput**: Vectors/second ingested.
- **Replication lag**: Bytes behind primary.

### Operational Runbooks
- **High latency**: Check ef_search parameter, shard balance, compaction lag.
- **Low recall**: Increase ef_search, rebuild index with higher ef_construction, check for stale segments.
- **OOM**: Offload cold collections to IVF-PQ, increase shard count.

## 13) Security (1-3 min)

- **Multi-tenant isolation**: Partition-level isolation. Queries can only access own partitions. Enforced in query coordinator.
- **Authentication**: API keys per tenant, mTLS between internal services.
- **Encryption**: Vectors encrypted at rest (AES-256). TLS 1.3 in transit.
- **Access control**: Per-collection RBAC (read, write, admin).
- **Audit log**: All operations logged with tenant, collection, and operation type.
- **Vector privacy**: Vectors are high-dimensional and not directly invertible, but combined with metadata they can be sensitive. Treat as PII where applicable.

## 14) Team and Operational Considerations (1-2 min)

- **Core Engine Team** (4-5 engineers): Index structures (HNSW, IVF-PQ), search algorithms, SIMD kernels.
- **Storage Team** (3-4 engineers): WAL, segment management, compaction, replication.
- **Distributed Systems Team** (3-4 engineers): Sharding, routing, consensus, rebalancing.
- **API/SDK Team** (2-3 engineers): Query language, client libraries, bulk import.
- **SRE** (2-3 engineers): Cluster operations, capacity planning, performance tuning.

## 15) Common Follow-up Questions

1. **How do you handle dimensionality changes (model upgrade)?**
   - Create a new collection with new dimension. Re-embed and migrate data. Alias the old collection name to the new one after migration. Keep old collection until confirmed.

2. **How do you support multi-vector search (e.g., text + image embeddings)?**
   - Store multiple vector fields per record. Support weighted combination of distances at query time.

3. **How do you handle vectors that are very similar (near-duplicates)?**
   - Deduplication at insert time using hash-based detection (SimHash). Optional dedup pass during compaction.

4. **How do you guarantee consistency between vector index and metadata index?**
   - Both updated from the same WAL entry atomically within a shard. Cross-shard operations use two-phase commit.

5. **How do you benchmark and tune recall vs. latency?**
   - Golden dataset of queries with known exact KNN results. Sweep ef_search values, plot recall vs. latency curve. Choose operating point per collection SLA.

## 16) Closing Summary (30-60s)

"We designed a distributed vector database that stores 5 billion vectors across horizontally sharded nodes, each running an HNSW index for sub-10ms ANN queries with over 95% recall. The system handles hybrid metadata filtering through a selectivity-aware strategy that chooses between pre-filtering, post-filtering, and bitmap-guided search. Writes go through a replicated WAL with LSM-style segment management, enabling real-time insert visibility. The architecture scales to hundreds of billions of vectors by tiering hot data on HNSW in-memory indexes and cold data on disk-backed IVF-PQ, with GPU-accelerated batch search at the highest scale."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| M (HNSW edges) | 32 | Higher = better recall + more memory |
| ef_construction | 400 | Higher = better index quality, slower build |
| ef_search | 128 | Per-query; higher = better recall, slower search |
| nprobe (IVF) | 32 | More probes = better recall, slower search |
| PQ subspaces | 64 | More = better compression, slightly lower accuracy |
| shard_size | 50M vectors | Balance memory per shard vs. number of shards |
| wal_sync_mode | fsync | async for throughput, fsync for durability |
| replication_factor | 3 | Balance durability vs. storage cost |
| memtable_threshold | 100K vectors | Controls flush frequency |
| compaction_interval | 1 hour | Frequency of segment merges |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| ANN | Approximate Nearest Neighbor -- trade exact results for speed |
| HNSW | Hierarchical Navigable Small World -- graph-based ANN index |
| IVF | Inverted File Index -- cluster-based ANN partitioning |
| PQ | Product Quantization -- vector compression technique |
| Recall@K | Fraction of true top-K results found by ANN |
| ef_search | HNSW search exploration factor (recall/speed knob) |
| Cosine similarity | Angle-based distance; 1 = identical direction |
| L2 distance | Euclidean distance between vectors |
| WAL | Write-Ahead Log -- ensures durability before acknowledging writes |
| Segment | Immutable batch of vectors flushed from memory to disk |
| Tombstone | Marker for a deleted vector, cleaned up during compaction |

## Appendix C: References

- Malkov & Yashunin, "Efficient and robust approximate nearest neighbor using HNSW graphs" (2018)
- Jegou et al., "Product Quantization for Nearest Neighbor Search" (2011)
- FAISS library (Facebook AI Similarity Search)
- Milvus architecture documentation
- Weaviate and Qdrant design documents
- Pinecone engineering blog
