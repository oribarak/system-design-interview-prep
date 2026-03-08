# Vector Database -- Cheat Sheet

## Key Numbers
- 5B vectors, 1024 dimensions, 10K QPS, < 10ms p99 latency
- Recall@10 > 95% with HNSW index
- Storage: 10 TB (FP16 vectors) + 1 TB (HNSW graph) + 500 GB (metadata)
- 100 shards x 50M vectors each, 3x replication
- HNSW per-vector overhead: ~2.2 KB (FP16 vector + graph edges)
- IVF-PQ compression: 64 bytes/vector (32x compression)

## Core Components
Query Coordinator (scatter-gather), Write Coordinator (hash routing), Shard Nodes (WAL + MemTable + HNSW index + metadata index), Cluster Controller (rebalancing), Index Builder, Compaction Service, Bulk Import Service, etcd (topology)

## Architecture in One Sentence
Vectors are hash-sharded across nodes, each maintaining an HNSW index in memory with WAL-backed durability and RocksDB metadata indexes, queried via scatter-gather with selectivity-aware hybrid filtering.

## Top 3 Trade-offs
1. **HNSW vs. IVF-PQ**: HNSW gives best recall and latency but uses 30x more memory; IVF-PQ compresses to 64 bytes/vector for cold data.
2. **Pre-filter vs. post-filter**: Post-filter is simpler but wastes search effort on selective filters; hybrid with selectivity estimation handles all cases.
3. **Sync vs. async replication**: Sync gives strong consistency but doubles write latency; async with read-your-writes is the practical middle ground.

## Scaling Story
- 1x: 100 shards, HNSW in memory, 3x replication.
- 10x: Tiered storage (hot HNSW, warm IVF-PQ on NVMe), partition pruning.
- 100x: GPU-accelerated batch search, disaggregated compute/storage, learned quantization, hierarchical coarse-to-fine search.

## Closing Statement
"The database shards 5B vectors across HNSW-indexed nodes with WAL-backed durability, serves 10K QPS at sub-10ms p99 with 95%+ recall, and uses selectivity-aware hybrid filtering for combined vector-metadata queries."
