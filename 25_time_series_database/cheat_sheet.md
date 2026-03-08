# Time-Series Database — Cheat Sheet

## Key Numbers
- Write QPS: 5M data points/sec, ~200 MB/s ingestion
- Storage: ~1.7 TB/day compressed (10:1 ratio via Gorilla encoding)
- 30-day raw retention: ~51 TB; 1-year downsampled: ~500 GB
- Read QPS: 50K queries/sec; p99 query latency < 200 ms
- Active series cardinality: 10 million

## Core Components (one line each)
- **Write Gateway**: validates, routes by series hash to ingest nodes
- **Ingest Node**: WAL + in-memory head block, flushes 2-hour immutable blocks
- **Series Index**: inverted index mapping label pairs to postings lists (roaring bitmaps)
- **Compactor**: merges small blocks into larger ones, applies downsampling
- **Query Node**: reads index, locates blocks, decompresses chunks, aggregates
- **Object Storage**: durable long-term home for compressed blocks
- **Retention Enforcer**: deletes blocks past retention cutoff

## Architecture in One Sentence
Append-only WAL-backed ingest nodes compress data via Gorilla encoding into time-partitioned immutable blocks, sharded by series hash and queried through an inverted label index.

## Top 3 Trade-offs
1. Push vs. pull ingestion: push handles ephemeral high-cardinality targets better; pull simplifies discovery
2. Compression vs. query speed: Gorilla encoding saves 10x storage but requires decompression on every read
3. Cardinality limits vs. flexibility: capping unique series prevents index explosion but restricts use cases

## Scaling Story
- **10x**: more ingest nodes, separate read/write paths, tiered storage (SSD + S3), parallel compaction
- **100x**: regional clusters, Kafka write buffer, aggressive multi-level downsampling, query pushdown to storage nodes

## Closing Statement
A TSDB optimized for append-heavy, range-query workloads using Gorilla compression, time-partitioned blocks, inverted label indexing, and horizontal sharding by series hash.
