# Search Autocomplete — Cheat Sheet

## Key Numbers
- 5 B searches/day, ~116K autocomplete QPS (debounced keystrokes)
- 100 M unique queries in trie
- Trie size: ~10-20 GB compressed (fits in RAM)
- Target p99 latency: < 50 ms
- Top 10 suggestions per prefix, ~500 B per response

## Core Components
- **Suggestion Server**: Holds compressed trie (radix tree) in RAM, O(L) prefix lookup
- **Trie Builder**: Offline job building trie from aggregated frequencies with pre-computed top-K
- **Flink Streaming**: 5-min trending trie updates for breaking queries
- **Spark Batch**: Hourly full frequency aggregation with decay weighting
- **S3 Snapshot Store**: Versioned serialized trie binaries for blue-green deploy
- **Blocklist Filter**: Removes offensive queries at build and serve time

## Architecture in One Sentence
An offline Spark/Flink pipeline aggregates search frequencies and builds a compressed in-memory trie with pre-computed top-K at each node, served by replicated servers with O(L) prefix lookups and blue-green trie swaps.

## Top 3 Trade-offs
1. **Trie vs Elasticsearch**: Custom trie gives sub-ms lookups; ES is easier but 10-20x slower
2. **Pre-computed vs on-the-fly top-K**: Pre-computed is O(L) but stale until rebuild; on-the-fly is O(N) but always fresh
3. **Full rebuild vs incremental update**: Full rebuild is simpler and guarantees consistency; incremental is faster but complex to implement correctly

## Scaling Story
- **10x**: Replicate trie servers (20-50 replicas), shard by prefix range, CDN-cache popular prefixes
- **100x**: Push trie snapshots to CDN edge, client-side caching of prefix hierarchies, regional trending tries

## Closing Statement
The autocomplete system's key insight is pre-computing top-K suggestions at every trie node during offline build, turning each prefix lookup into an O(L) traversal. A two-speed pipeline (hourly batch + minute-level streaming) balances freshness with build cost.
