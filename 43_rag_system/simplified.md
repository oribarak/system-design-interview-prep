# RAG System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a system where users ask natural language questions over a large document corpus and get grounded, cited answers. The hard parts are achieving high retrieval recall across 500M chunks, preventing the LLM from hallucinating beyond what the retrieved documents actually say, and keeping end-to-end latency under 3 seconds.

## 2. Requirements
**Functional:** Ingest documents (PDF, HTML, text) with automatic chunking and embedding. Answer queries with cited, grounded responses. Multi-turn conversation support. Multi-tenant document isolation. Metadata filtering (date, source, tags).

**Non-functional:** End-to-end query latency under 3 seconds p95. Retrieval recall@20 above 90%. Hallucination rate under 5%. Ingestion throughput of 100K documents/day. 99.9% availability.

## 3. Core Concept: Hybrid Retrieval with Cross-Encoder Re-Ranking
Neither vector search nor keyword search alone is sufficient. Vector search finds semantically similar content but misses exact terms; keyword search (BM25) catches exact names and numbers but misses paraphrases. We combine both using Reciprocal Rank Fusion (RRF): documents appearing in both result lists get boosted scores. A cross-encoder re-ranker then scores the top 100 merged candidates by examining each query-chunk pair together, improving precision@10 by 15-25%. Only the top 10 re-ranked chunks go to the LLM for grounded answer generation with citation markers.

## 4. High-Level Architecture
```
Documents --> Ingestion Pipeline (parse, chunk, embed) --> Vector DB + Text Index
                                                                  |
User Query --> Embedding Service --> Hybrid Retriever (vector + BM25 + RRF)
                                          |
                                    Re-Ranker (cross-encoder, top 100 -> top 10)
                                          |
                                    Prompt Builder (context + history)
                                          |
                                    LLM Service --> Citation Verifier --> Response
```
- **Hybrid Retriever**: Combines vector search (HNSW index) and BM25 keyword search via RRF.
- **Re-Ranker**: Cross-encoder model scores query-chunk pairs for precise relevance (~200ms).
- **Prompt Builder**: Inserts top 10 chunks as context with instructions to cite sources.
- **Citation Verifier**: NLI model checks if each cited claim is actually supported by the source chunk.

## 5. How a Query Gets Answered
1. User sends a question; query rewriter expands it with conversation context if multi-turn.
2. Embedding Service encodes the question into a vector.
3. Vector search returns top 100 by cosine similarity; BM25 returns top 100 by keyword match.
4. RRF merges and ranks the combined 200 candidates (with dedup).
5. Cross-encoder re-ranks the top 100 merged results; selects the top 10.
6. Prompt Builder inserts the 10 chunks as numbered context into the LLM prompt.
7. LLM generates an answer with [1], [2] citation markers referencing specific chunks.
8. Citation Verifier uses an NLI model to check each claim against its cited chunk; flags unsupported claims.

## 6. What Happens When Things Fail?
- **LLM service failure**: Retry with backoff. Fall back to a smaller model. In degraded mode, return the retrieved chunks directly without a generated answer.
- **Vector DB failure**: Read from replica on primary failure. Rebuild index from stored embeddings if needed.
- **Low retrieval confidence**: If no retrieved chunk scores above a threshold, return "I don't have enough information" rather than guessing.

## 7. Scaling
- **10x**: Shard vector DB across multiple nodes. Cache frequent queries and their retrieved chunks in Redis. Batch embedding calls for ingestion.
- **100x**: Tiered retrieval with compressed vectors (PQ) for coarse search, then full vectors for top candidates. Model distillation for a faster re-ranker. Semantic caching to reuse answers for similar queries.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Hybrid retrieval (vector + BM25) vs. vector-only | 10-20% higher recall, but requires maintaining two index systems and a fusion strategy |
| Cross-encoder re-ranking | 15-25% precision improvement, but adds ~200ms latency to the query path |
| NLI citation verification | Catches hallucinated claims before they reach the user, but adds post-processing latency and may occasionally reject valid claims |

## 9. Closing (30s)
> We designed a RAG system with hybrid retrieval combining vector search and BM25 via reciprocal rank fusion, cross-encoder re-ranking for precision, and grounded generation with NLI-based citation verification to minimize hallucinations. The system processes 50M documents across 1,000 tenants, answers queries in under 3 seconds with cited sources, and scales through sharded vector indexes and tiered retrieval.
