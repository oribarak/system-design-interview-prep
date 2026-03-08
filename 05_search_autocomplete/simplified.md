# Search Autocomplete -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a system that suggests the top matching queries as a user types in a search box. The challenge is returning relevant suggestions in under 50ms at 100K+ QPS, while keeping suggestions fresh with trending queries that emerge in real-time.

## 2. Requirements
**Functional:** Given a prefix string, return the top 10 most popular query completions. Suggestions are ranked by search frequency with recency weighting. Update suggestions periodically to reflect trending queries. Filter offensive content.

**Non-functional:** Sub-50ms p99 latency, 99.99% availability, 100K+ QPS, eventually consistent (suggestions can lag trends by minutes), graceful degradation (show no suggestions rather than wrong ones).

## 3. Core Concept: Trie with Pre-Computed Top-K at Every Node
The key insight is pre-computing the top-K suggestions at every node in the trie during build time. A naive approach would traverse all descendants on each query -- far too slow for short prefixes like "a" with millions of completions. Instead, during offline trie construction, we bubble up the K most popular completions to every node. At query time, we just walk to the prefix node in O(L) steps (L = prefix length) and return its stored list. The trie fits in RAM (10-20 GB compressed) and is rebuilt periodically from aggregated search logs.

## 4. High-Level Architecture

```
[ Search Logs ] --> [ Kafka ] --> [ Aggregation (Spark/Flink) ] --> [ Trie Builder ]
                                                                         |
                                                                         v
                                                                   [ S3 Snapshot ]
                                                                         |
                                                                         v
Client --> [ Load Balancer ] --> [ Suggestion Servers (in-memory trie) ]
```

- **Aggregation Pipeline**: Consumes search logs from Kafka, counts query frequencies by time bucket, outputs aggregated data.
- **Trie Builder**: Offline job that reads aggregated frequencies, builds a trie with pre-computed top-K per node, serializes to S3.
- **Suggestion Servers**: Stateful servers holding the trie in memory. Multiple replicas for throughput and availability.
- **S3**: Stores serialized trie snapshots. Servers pull the latest on startup or periodic refresh.

## 5. How a Suggestion Request Works
1. User types a character. The client debounces (waits 200ms) before sending a request.
2. Request hits the load balancer, which routes to a suggestion server.
3. The server walks the in-memory trie to the node matching the prefix.
4. Returns the pre-stored top-K list from that node -- no further computation needed.
5. For freshness, results from the main trie are merged with a smaller "trending trie" built from the last 1-2 hours of data.
6. Offensive queries are filtered at both build time and serving time.

## 6. What Happens When Things Fail?
- **Suggestion server crashes**: Replicas handle traffic. Load balancer health checks remove failed nodes. Users see no interruption.
- **Trie build fails**: Servers continue serving the previous snapshot. Suggestions become stale but remain valid. Alert on trie age.
- **Entire suggestion system is down**: The search box still works -- users just type the full query without suggestions. No core functionality is lost.

## 7. Scaling
- **10x (1.2M QPS)**: Replicate trie servers to 20-50 instances. Shard by prefix range if needed (server A handles a-m, server B handles n-z). Cache popular prefix responses at the CDN edge.
- **100x (12M QPS)**: Push trie snapshots to CDN edge locations and serve suggestions from the edge. Client-side caching so typing "how to" reuses the response from "how t". Regional tries for location-specific trending queries.

## 8. Key Trade-offs to Mention

| Decision | Trade-off |
|---|---|
| Pre-computed top-K at each node vs on-the-fly ranking | Pre-computed gives O(L) lookups but suggestions are stale until next rebuild. On-the-fly is flexible but O(N) per query. |
| Full trie rebuild (hourly) + trending trie (every 5 min) | Two-speed architecture balances long-term popularity with breaking trends, at the cost of maintaining two data structures. |
| In-memory trie vs Elasticsearch completion suggester | Custom trie gives sub-millisecond lookups. Elasticsearch is easier to set up but 10-20x higher latency. Custom trie is necessary at scale. |

## 9. Closing (30s)
> "This autocomplete system uses an in-memory trie with pre-computed top-K suggestions at each node, enabling O(L) prefix lookups with sub-50ms latency. The trie is built offline from aggregated search logs via a Spark/Flink pipeline and deployed as snapshots to replicated servers. A two-speed architecture provides hourly full rebuilds for long-term popularity and a minute-level trending trie for breaking queries. The system degrades gracefully -- if suggestions are unavailable, the search box still works."
