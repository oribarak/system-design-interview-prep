# Search Engine -- Cheat Sheet

## Key Numbers
- 50B pages indexed, ~250 TB inverted index, 2.5 PB raw corpus
- 100K QPS peak, p50 < 200 ms, p99 < 1 s
- ~500 index shards, 3 replicas each, ~100M docs/shard
- Cache hit rate ~60% (Zipf query distribution)

## Core Components
- URL Frontier + Crawler Fleet: discover and fetch web pages
- Document Store: raw page content and metadata
- Batch/Streaming Indexer: builds inverted index segments
- Index Shards: document-partitioned, replicated inverted indexes
- Query Broker: fans out queries to all shards, merges results
- Multi-Stage Ranker: BM25 retrieval -> lightweight ML -> heavy transformer re-rank
- Snippet Generator: extracts query-relevant text from stored content

## Architecture in One Sentence
Crawlers feed pages into a document store; batch and streaming indexers build document-partitioned inverted indexes across hundreds of shards; a query broker fans out queries, retrieves BM25 candidates, applies multi-stage ML ranking, and returns snippeted results in under 200 ms.

## Top 3 Trade-offs
1. Document-partitioned vs. term-partitioned index: chose document-partitioned for simpler query execution despite full fan-out cost.
2. Multi-stage ranking cascade vs. single model: cascade meets latency budget by limiting expensive GPU inference to top-100 candidates.
3. Hybrid batch/streaming indexing vs. pure batch: streaming adds complexity but enables minute-level freshness for breaking news.

## Scaling Story
10x: more shard replicas, larger cache layer, GPU clusters for ranking.
100x: multi-region with full index replicas, aggressive tiered indexing, pre-computed results for top queries, custom hardware for inference.

## Closing Statement
A document-partitioned inverted index with multi-stage ranking, tiered indexing for freshness, and horizontal scaling via shard replication delivers sub-second search across 50 billion web pages at 100K QPS.
