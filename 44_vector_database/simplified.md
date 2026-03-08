# Vector Database -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a database that stores billions of high-dimensional embedding vectors and supports fast approximate nearest neighbor (ANN) search with metadata filtering. The hard parts are achieving sub-10ms query latency at 95%+ recall across 5 billion vectors, handling real-time inserts alongside searches, and combining vector similarity with metadata predicates efficiently.

## 2. Requirements
**Functional:** Insert, update, and delete vectors with metadata. Bulk import millions of vectors. ANN search with configurable top-K and recall/speed trade-off. Support cosine, L2, and inner product distances. Combined metadata filtering with vector search. Multi-collection support.

**Non-functional:** Query latency under 10ms p99 at 10K QPS. Recall@10 above 95%. Support 5B vectors with horizontal scaling. Real-time insert visibility under 1 second. 99.95% availability with no data loss.

## 3. Core Concept: HNSW Index with Selectivity-Aware Filtering
HNSW (Hierarchical Navigable Small World) is a multi-layer graph where top layers have sparse long-range connections and the bottom layer is dense. Search starts at the top, greedily descends, then expands at the bottom with an exploration factor (ef_search) that controls the recall/speed trade-off. For metadata filtering, we estimate filter selectivity: if the filter matches more than 10% of data, post-filter with oversampling; if 1-10%, use bitmap-guided HNSW traversal that skips non-matching nodes; if under 1%, brute-force the small filtered set.

## 4. High-Level Architecture
```
Client --> Query Coordinator (scatter-gather) --> Shard Nodes (HNSW index + metadata index)
                                                       |
       Write Coordinator --> WAL --> MemTable --> Flush to Segments
                                                       |
                                                 Compaction Service
                                                       |
                                              Replication Manager (3x)
```
- **Query Coordinator**: Parses queries, routes to relevant shards, merges top-K results.
- **Shard Nodes**: Each stores ~50M vectors with an HNSW index in memory and metadata B-tree indexes in RocksDB.
- **Write Path**: WAL for durability, MemTable for buffering, periodic flush to immutable segments.
- **Compaction Service**: Merges segments, garbage-collects tombstoned vectors.

## 5. How a Filtered Search Works
1. Client sends a query vector with filter: "category = electronics AND price < 500."
2. Query Coordinator estimates filter selectivity from metadata index statistics.
3. If selectivity is 5% (medium): use bitmap-guided HNSW search.
4. Coordinator scatters the query to all relevant shards.
5. Each shard traverses its HNSW graph, skipping nodes that fail the bitmap check.
6. Each shard returns its local top-K results with scores and metadata.
7. Coordinator merges results from all shards, re-ranks, and returns the global top-K.

## 6. What Happens When Things Fail?
- **Shard failure**: Each shard replicated 3x with Raft consensus. Primary failure triggers leader election among replicas with no data loss.
- **Index corruption**: Rebuild from WAL and segment files. Checksums on all persisted data detect corruption early.
- **Node failure**: Controller detects via heartbeat timeout. Replicas promoted; new replicas created on healthy nodes via segment copy and WAL replay.

## 7. Scaling
- **10x (50B vectors)**: 1,000 shards across 200+ nodes. Tiered storage with hot shards (HNSW in memory) and warm shards (IVF-PQ on NVMe). Partition pruning to skip irrelevant shards.
- **100x (500B vectors)**: GPU-accelerated batch search with FAISS. Disaggregated compute and storage. Hierarchical search with a coarse global index to identify candidate shards.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| HNSW (in-memory graph) vs. IVF-PQ (disk-based) | HNSW gives best recall and latency but requires ~2.2 KB/vector in RAM; IVF-PQ compresses to 64 bytes but has lower recall |
| Selectivity-aware hybrid filtering | Adapts strategy to filter cardinality for best performance, but requires maintaining metadata statistics and adds query planning complexity |
| Async replication with read-your-writes | Lower write latency than synchronous replication, but replicas may serve slightly stale data for other clients |

## 9. Closing (30s)
> We designed a distributed vector database storing 5 billion vectors across sharded nodes, each running HNSW for sub-10ms ANN queries with 95%+ recall. Metadata filtering uses a selectivity-aware strategy choosing between pre-filter, post-filter, and bitmap-guided search. Writes go through a replicated WAL with LSM-style segment management for real-time insert visibility. The system scales to hundreds of billions of vectors through tiered storage with HNSW for hot data and IVF-PQ for cold data.
