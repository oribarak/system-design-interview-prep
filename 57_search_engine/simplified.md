# Search Engine (Google Search) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a web-scale search engine that crawls and indexes 50 billion web pages and serves keyword queries with sub-second latency at 100K QPS. The hard part is building and querying a 250TB inverted index fast enough, ranking results using both text relevance (BM25) and authority signals (PageRank), and keeping results fresh for breaking news while handling the long tail of the web.

## 2. Requirements
**Functional:** Crawl and index web pages continuously. Build an inverted index mapping terms to documents. Parse queries, retrieve matching documents, rank by relevance, and return top results with snippets. Support autocomplete and spelling correction.

**Non-functional:** p50 under 200ms, p99 under 1s for queries. 100K QPS sustained. 99.99% query serving availability. Breaking news indexed within minutes. High precision and recall in top-10 results.

## 3. Core Concept: Document-Partitioned Inverted Index with Multi-Stage Ranking
The inverted index maps each term to a posting list of document IDs where that term appears. The index is sharded by document (each shard holds a random subset of documents with a complete inverted index for those documents). Every query fans out to all 500 shards in parallel. Each shard intersects posting lists using skip pointers, scores candidates with BM25, and returns its top-K. The broker merges per-shard results into the global top-K. A multi-stage ranker then re-ranks: BM25 first (top 1,000), then a lightweight ML model (top 200), then a transformer-based heavy ranker (top 100).

## 4. High-Level Architecture
```
Query --> [Query Parser] --> [Query Broker] -- fans out to all shards
                                  |
              [Index Shard 1] [Index Shard 2] ... [Index Shard 500]
                                  |
              [Merge + Re-Rank] -- BM25 -> lightweight ML -> heavy ML
                                  |
              [Snippet Generator] --> Results

Crawl: [URL Frontier] --> [Crawler Fleet] --> [Document Store] --> [Indexer]
```
- **Query Broker**: Fans out parsed query to all 500 index shards in parallel, merges top-K results.
- **Index Shards**: Each holds a complete inverted index for a subset of documents. 3x replicated.
- **Multi-Stage Ranker**: BM25 (index-time), lightweight ML (under 10ms), transformer re-ranker (under 50ms on GPU).
- **Indexer**: Batch MapReduce for bulk corpus, streaming jobs for near-real-time freshness.

## 5. How a Search Query Works
1. User submits query "best Italian restaurants NYC."
2. Query Parser tokenizes and stems: [best, italian, restaurant, nyc].
3. Query Broker fans out to all 500 shards in parallel.
4. Each shard retrieves posting lists for each term, intersects using skip pointers.
5. BM25 scoring produces per-shard top-1,000 candidates.
6. Broker merges results, feeds top-1,000 to lightweight ML ranker (adds PageRank, freshness, domain authority).
7. Top-100 go to heavy transformer ranker (cross-attention on query-document pairs).
8. Snippet Generator extracts relevant text passages from stored documents.

## 6. What Happens When Things Fail?
- **Index shard unavailable**: Return partial results from remaining shards with a quality indicator. Each shard has 3 replicas across failure domains.
- **Heavy ML ranker overloaded**: Circuit breaker falls back to lightweight ranker only. Quality degrades slightly but latency is maintained.
- **Index data corruption**: New index versions are built offline and swapped in atomically (blue-green deployment). Rollback to previous version if quality drops.

## 7. Scaling
- **10x**: Increase to 5,000 shards with more replicas. Cache hit rate exceeds 60% for popular queries (Zipf distribution). GPU clusters for heavy ranker with batched inference.
- **100x**: Multi-region deployment with full index replicas per region. Pre-compute results for top 10,000 queries (covers 15% of traffic). Aggressive tiered indexing: Tier 1 (top 1B pages) searched for 90% of queries.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Document-partitioned vs term-partitioned sharding | Document-partitioned requires fan-out to all shards but avoids hot spots on popular terms |
| Multi-stage ranking cascade | Latency budget requires cheap first pass on many docs, expensive re-rank on few; single model cannot score all candidates |
| Streaming + batch hybrid indexing | Freshness for news (minutes) while batch handles efficiency on the bulk corpus |

## 9. Closing (30s)
> Three pipelines: a distributed crawler feeding a document store, a hybrid batch/streaming indexer building document-partitioned inverted indexes across 500 shards, and a query-serving layer that fans out to all shards, retrieves candidates via BM25, and applies multi-stage ML ranking -- all within 200ms p50 at 100K QPS. The key insight is the ranking cascade: cheap BM25 on millions, lightweight ML on thousands, expensive transformer on hundreds, each stage progressively trading compute for precision.
