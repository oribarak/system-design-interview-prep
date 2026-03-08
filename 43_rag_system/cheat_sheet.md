# RAG System -- Cheat Sheet

## Key Numbers
- 50M documents, 500M chunks, 1,000 tenants
- 167 QPS queries, 100K new documents/day ingested
- Vectors: 1024-dim, 1-2 TB total storage
- End-to-end latency: < 3s p95 (embedding 5ms + retrieval 15ms + re-rank 200ms + LLM 1.5s)
- Target retrieval recall@20 > 90%, hallucination rate < 5%

## Core Components
Document Parser, Hierarchical Chunker, Bi-Encoder Embedding Service, Vector DB (Milvus/HNSW), Elasticsearch (BM25), RRF Hybrid Merger, Cross-Encoder Re-Ranker, Query Rewriter, Prompt Builder, LLM Service, Citation Verifier (NLI), Conversation Manager

## Architecture in One Sentence
Documents are chunked and embedded into a vector DB and text index; queries undergo hybrid retrieval (vector + BM25 + RRF), cross-encoder re-ranking, and LLM generation with NLI-verified citations.

## Top 3 Trade-offs
1. **Hybrid vs. vector-only retrieval**: Hybrid adds BM25 for exact-match strength, improving recall 10-20% at the cost of maintaining a second index.
2. **Cross-encoder re-ranking vs. skip**: Adds ~200ms but improves precision@10 by 15-25%, critical for answer quality.
3. **Hierarchical vs. fixed-size chunking**: Hierarchical chunks at semantic boundaries produce better retrieval quality but require more complex parsing.

## Scaling Story
- 1x: Single Milvus cluster, Elasticsearch, and LLM service handles 167 QPS.
- 10x: Sharded vector DB, query/chunk caching in Redis, batched embedding.
- 100x: Tiered retrieval (PQ coarse + full vectors fine), distilled re-ranker, semantic answer cache, multi-region vector replicas.

## Closing Statement
"The system grounds LLM answers in retrieved documents via hybrid search with cross-encoder re-ranking, uses hierarchical chunking with parent expansion for rich context, and verifies citations with NLI to keep hallucination below 5%."
