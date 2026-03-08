# System Design Interview: Search Autocomplete — "Perfect Answer" Playbook

Use this as a speak-aloud script + reference architecture. Optimised for a 45-60 minute system design interview.

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a search autocomplete system that suggests the top matching queries as a user types. I'll clarify scope and scale, estimate traffic, design the API and data model, then deep dive on the trie data structure and ranking strategy, followed by the data collection pipeline and how suggestions stay fresh."

---

## 1) Clarifying Questions (2-5 min)

### Scope
- Are suggestions based only on **prefix matching** or also **substring/fuzzy matching**?
- Do we personalise suggestions per user or use **global popularity**?
- How many suggestions per keystroke? (typically 5-10)
- Do we support **multiple languages** and **multi-word queries**?
- Do we need **trending queries** (time-weighted popularity)?

### Scale
- How many **search queries per day**?
- How many **autocomplete requests per second**? (each keystroke triggers a request)
- How many **unique query strings** to index?

### Constraints
- Maximum acceptable latency for suggestions?
- Do we need to filter offensive/inappropriate suggestions?

> Default assumptions: prefix matching, global popularity with optional personalisation, top 10 suggestions, 5 B searches/day, ~60 K autocomplete QPS (each search = ~5 keystrokes, but debounced to ~1-2 requests), 100 M unique queries in the trie, p99 latency < 50 ms, English initially.

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|---|---|---|
| Searches/day | given | 5 B |
| Autocomplete QPS | 5 B * 2 keystrokes / 86,400 | ~116 K QPS |
| Unique queries | given | 100 M |
| Avg query length | assumed | 20 characters |
| Trie size (estimate) | 100 M queries * 20 chars * ~50 B/node overhead | ~100 GB (uncompressed) |
| Trie size (compressed) | with path compression, sharing | ~10-20 GB (fits in RAM) |
| Suggestion response size | 10 suggestions * 50 B avg | ~500 B per response |
| Bandwidth | 116 K * 500 B | ~58 MB/s |

> Key insight: The trie must fit in memory for sub-50ms latency. At 10-20 GB compressed, it fits on a single machine but should be replicated for availability and throughput.

---

## 3) Requirements (3-5 min)

### Functional
- Given a prefix string, return the top-K (10) most relevant query suggestions.
- Suggestions are ranked by popularity (search frequency), with optional recency weighting.
- Suggestions update periodically to reflect trending queries (not real-time).
- Filter offensive or blocked queries from suggestions.

### Non-functional
- **Low latency**: p99 < 50 ms for suggestion responses.
- **High availability**: 99.99% uptime (autocomplete is a core UX feature).
- **Scalable**: handle 100K+ QPS.
- **Eventually consistent**: suggestions can lag real-world trends by minutes to hours.
- **Fault tolerant**: degrade gracefully (show no suggestions rather than wrong ones).

---

## 4) API Design (2-4 min)

### Get Suggestions
```
GET /api/v1/suggestions?prefix=how+to&limit=10&lang=en
Response 200: {
    "prefix": "how to",
    "suggestions": [
        { "query": "how to tie a tie", "score": 98500 },
        { "query": "how to screenshot on mac", "score": 87200 },
        { "query": "how to cook rice", "score": 76100 },
        ...
    ]
}
```

### Report Search (Internal -- feeds the data pipeline)
```
POST /api/v1/searches
Body: { "query": "how to tie a tie", "user_id": "...", "timestamp": "..." }
Response 202 Accepted
```

### Update Suggestions (Internal -- trie rebuild trigger)
```
POST /internal/suggestions/rebuild
Body: { "data_source": "s3://aggregated-queries/2026-03-07/" }
Response 202 Accepted
```

---

## 5) Data Model (3-5 min)

### Trie Node (In-Memory)
| Field | Type | Notes |
|---|---|---|
| children | MAP[char -> TrieNode] | Child nodes |
| is_terminal | BOOL | Marks end of a valid query |
| top_k | LIST[(query, score)] | Pre-computed top-K suggestions for this prefix |

### Query Frequency Table (Offline Storage)
| Column | Type | Notes |
|---|---|---|
| query | VARCHAR | Normalised query string |
| frequency | BIGINT | Total search count |
| last_updated | TIMESTAMP | -- |
| language | VARCHAR(5) | en, es, fr, etc. |
| blocked | BOOL | Offensive content flag |

### Aggregated Query Logs
| Column | Type | Notes |
|---|---|---|
| query | VARCHAR | Normalised query string |
| time_bucket | TIMESTAMP | 1-hour or 1-day bucket |
| count | BIGINT | Searches in this bucket |

### Storage Choices
- **Trie**: In-memory on suggestion servers. Rebuilt periodically from aggregated data.
- **Query logs**: Kafka -> aggregation pipeline -> stored in HDFS/S3 as Parquet.
- **Frequency table**: Materialized in a data warehouse (BigQuery, Hive) or key-value store.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow

**Online path (serve suggestions)**: Client -> Load Balancer -> Suggestion Service (in-memory trie) -> return top-K.

**Offline path (build trie)**: Search logs -> Kafka -> Aggregation Pipeline (Spark/Flink) -> compute query frequencies -> build trie snapshot -> deploy to Suggestion Service.

### Components
| Component | Description |
|---|---|
| **Suggestion Service** | Stateful service holding the trie in memory. Handles prefix lookups. Multiple replicas for throughput. |
| **Load Balancer** | Routes autocomplete requests. Can shard by prefix range for large tries. |
| **Kafka** | Ingests raw search events from the search service. |
| **Aggregation Pipeline** | Spark/Flink job: dedup, count, aggregate query frequencies by time bucket. |
| **Trie Builder** | Offline job: reads aggregated frequencies, builds trie with pre-computed top-K, serializes to a snapshot file. |
| **Snapshot Store (S3)** | Stores serialized trie snapshots. Suggestion servers pull the latest snapshot on startup or refresh. |
| **Filter Service** | Checks queries against blocklist before insertion into trie. |

### One-sentence summary
> "An offline pipeline aggregates search frequencies and builds an in-memory trie with pre-computed top-K suggestions per prefix node, served by replicated stateful servers behind a load balancer."

---

## 7) Deep Dive #1: Trie Data Structure and Ranking (8-12 min)

### Why a Trie?
A trie (prefix tree) is the natural data structure for prefix matching. Given a prefix of length L, we reach the corresponding node in O(L) time. The challenge is efficiently returning the top-K most popular completions from that node.

### Naive Approach (Too Slow)
Traverse all descendants of the prefix node, collect all terminal nodes, sort by frequency, return top K. This is O(N) where N is the number of descendants -- too slow for popular short prefixes like "a" with millions of completions.

### Optimised: Pre-Computed Top-K at Each Node
During trie construction, pre-compute and store the top-K suggestions at every node in the trie. At query time, a prefix lookup is O(L) -- just traverse to the node and return its pre-stored top-K list.

**Build-time algorithm:**
```
def build_top_k(node):
    candidates = []
    if node.is_terminal:
        candidates.append((node.query, node.score))
    for child in node.children.values():
        candidates.extend(build_top_k(child))
    node.top_k = heapq.nlargest(K, candidates, key=lambda x: x.score)
    return node.top_k
```

**Trade-off**: This increases memory (each node stores K suggestions), but makes lookups O(L) instead of O(N). For K=10 and 20-char average query length, the memory overhead is ~200 bytes/node -- manageable.

### Path Compression (Compact Trie / Radix Tree)
Many trie paths have single-child chains (e.g., "q" -> "u" -> "e" -> "r" -> "y"). Compress these into single nodes storing the common prefix string. This reduces node count by 5-10x and cuts memory proportionally.

### Scoring and Ranking
The `score` for a query combines:
- **Frequency**: raw search count (dominant signal).
- **Recency**: exponential decay -- recent searches count more. `score = count * decay^(age_in_hours)`.
- **Trending bonus**: queries with rapidly increasing frequency get a boost.
- **Quality signals**: click-through rate on the suggestion, query length (shorter preferred).

### Trie Sharding (for very large tries)
If the trie is too large for a single machine (> 64 GB):
- Shard by prefix range: server A handles a-m, server B handles n-z.
- The load balancer routes based on the first character of the prefix.
- Each shard is replicated for availability.

---

## 8) Deep Dive #2: Data Collection and Freshness (5-8 min)

### The Freshness Problem
Users expect trending queries to appear in suggestions quickly (within minutes for breaking news). But rebuilding a 10 GB trie from scratch takes minutes to hours.

### Two-Speed Architecture

**Slow path (hourly/daily)**: Full trie rebuild from the complete aggregated frequency dataset. This captures the long-term popularity landscape.

**Fast path (every few minutes)**: A smaller "trending trie" built from the last 1-2 hours of search data. This captures breaking trends.

**Serving**: The suggestion service merges results from both tries at query time. The trending trie boosts scores for recently popular queries. If a query appears in both, take the higher score.

### Data Pipeline Flow
```
1. User searches -> search service logs event to Kafka.
2. Kafka -> Flink streaming job: aggregate counts per (query, 5-min bucket).
3. Every hour: Spark batch job reads from Kafka/HDFS, computes global frequencies.
4. Trie Builder reads batch output, builds full trie, writes snapshot to S3.
5. Suggestion servers detect new snapshot, load it (hot swap via blue-green).
6. Trending trie: Flink reads last 2 hours, builds mini-trie, pushes to servers every 5 min.
```

### Handling Offensive Content
- Maintain a blocklist of offensive queries (curated + ML-flagged).
- Filter at two points: (1) during trie build (exclude blocked queries), (2) at serving time (runtime filter for newly blocked queries before next trie rebuild).

---

## 9) Trade-offs and Alternatives (3-5 min)

### Trie vs Elasticsearch Prefix Queries
- **Trie**: Purpose-built for prefix completion. O(L) lookups. Pre-computed top-K. Sub-millisecond latency.
- **Elasticsearch completion suggester**: Easier to set up, supports fuzzy matching. But higher latency (5-20 ms) and harder to tune ranking.
- At Google/Bing scale, a custom trie is necessary. For smaller products, Elasticsearch is a pragmatic choice.

### In-Memory vs Disk-Based
- **In-memory trie**: Sub-millisecond lookups. Limited by RAM (10-64 GB per node).
- **Disk-based (RocksDB, LMDB)**: Larger capacity but 10-100x higher latency. Acceptable if p99 target is > 10 ms.

### Pre-Computed vs On-the-Fly Ranking
- **Pre-computed top-K** (our approach): O(L) query time, but stale until next rebuild.
- **On-the-fly ranking**: Flexible but O(N) per query. Only viable with a secondary index (inverted index by score).

### Personalization
- **Global suggestions** (our baseline): Same for all users. Simpler.
- **Personalized**: Augment with user's search history. Merge user's recent queries into the suggestion list with a personal boost. Requires per-user state (small -- last 100 queries).

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (1.2 M autocomplete QPS)
- **Replicate trie servers**: 20-50 replicas behind a load balancer.
- **Shard by prefix**: Split trie into 26 shards (a-z) or by frequency-weighted ranges.
- **CDN caching**: Cache popular prefix responses at the edge. The top 1000 prefixes cover a large fraction of traffic.

### At 100x (12 M autocomplete QPS)
- **Edge-deployed tries**: Push trie snapshots to CDN edge locations. Serve suggestions from the edge with sub-5ms latency.
- **Client-side caching**: Cache recent prefix responses in the browser/app. "how t" response includes suggestions for "how to" -- the client can show them locally as the user types the next character.
- **Hierarchical tries**: Global trie + regional tries (trending queries vary by region).

---

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure | Mitigation |
|---|---|
| Suggestion server crash | Replicas handle traffic. Load balancer health checks remove failed nodes. |
| Trie build fails | Serve the previous snapshot. Alert on staleness. |
| Kafka/pipeline failure | Trie is not updated but still serves. Suggestions become stale but remain valid. |
| Corrupted trie snapshot | Checksum validation before loading. Fall back to previous known-good snapshot. |

### Graceful Degradation
- If all suggestion servers are down, return empty suggestions. The search box still works -- users just type the full query.
- If latency exceeds threshold, return partial results (fewer than K suggestions) or empty.

### Blue-Green Trie Swap
- New trie is loaded into memory alongside the current one.
- Once fully loaded and validated, traffic is switched atomically.
- Old trie is deallocated. Zero downtime during updates.

---

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Suggestion latency**: p50/p95/p99 (target: p99 < 50 ms).
- **Trie freshness**: age of the current trie snapshot.
- **Suggestion coverage**: % of prefixes that return at least 1 suggestion.
- **Click-through rate**: % of displayed suggestions that are clicked.
- **Trie size**: node count, memory usage.
- **Pipeline lag**: Kafka consumer lag, batch job duration.

### Alerting
- Trie age > 2 hours (freshness SLA breach).
- Suggestion latency p99 > 100 ms.
- Trie build job fails 2 consecutive runs.
- Suggestion click-through rate drops by > 20% (quality regression).

---

## 13) Security (1-3 min)

- **Offensive content filtering**: Blocklist enforced at trie build and serving time. ML-based detection for new offensive queries.
- **Rate limiting**: Per IP / per user to prevent scraping the suggestion index.
- **Privacy**: Raw search logs are aggregated and anonymised before analysis. Individual user queries are not stored long-term unless opted in.
- **No PII in suggestions**: Ensure suggestions never expose personal information (phone numbers, emails, etc.).
- **HTTPS**: All autocomplete traffic encrypted.

---

## 14) Team and Operational Considerations (1-2 min)

- **Team**: 3-5 engineers. 1 for suggestion serving infrastructure, 1-2 for data pipeline (Kafka, Spark, Flink), 1 for ranking/quality, 1 for content moderation.
- **Deployment**: Trie servers as StatefulSets (each holds a trie in memory). Pipeline jobs in Spark/Flink on a shared data platform.
- **Rollout**: New trie snapshots deployed via blue-green swap. Ranking changes A/B tested on a subset of traffic.
- **On-call**: Alert on trie freshness, latency, pipeline failures.

---

## 15) Common Follow-up Questions

### "How do you handle typos / fuzzy matching?"
Add a separate spell-correction layer. For each prefix, generate edit-distance-1 variants and look up their top-K. Merge corrected suggestions with exact-match suggestions, clearly marking "did you mean..." alternatives. Libraries like SymSpell provide efficient fuzzy lookup.

### "How do you handle multi-word queries?"
Treat the entire query as a string. The trie indexes full query strings, not individual words. "how to" is a prefix that matches "how to cook rice", "how to tie a tie", etc. Word boundaries are just characters in the trie.

### "How do you prevent suggestion bombing (SEO manipulation)?"
Require a minimum frequency threshold before a query appears in suggestions. Apply velocity checks -- a query that spikes from 0 to 100K in an hour is suspicious. Use click-through rate as a quality signal (bot-generated searches won't have real clicks).

---

## 16) Closing Summary (30-60s)

> "This autocomplete system uses an in-memory trie with pre-computed top-K suggestions at each node, enabling O(L) prefix lookups with sub-50ms p99 latency. The trie is built offline by a Spark/Flink pipeline that aggregates search frequencies from Kafka. A two-speed architecture provides hourly full rebuilds for long-term popularity and minute-level trending trie updates for breaking queries. The system scales via trie replication, prefix-based sharding, and edge deployment. Offensive content is filtered at both build time and serving time."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|---|---|---|
| `top_k` | 10 | Suggestions per prefix |
| `min_frequency` | 100 | Minimum searches to appear in trie |
| `trie_rebuild_interval` | 1 hour | Full trie rebuild frequency |
| `trending_update_interval` | 5 min | Fast-path trending trie refresh |
| `decay_factor` | 0.95 per hour | Recency weighting |
| `max_prefix_length` | 50 chars | Ignore very long prefixes |
| `debounce_interval` | 200 ms | Client-side keystroke debounce |
| `cache_ttl_popular` | 5 min | CDN/edge cache for top prefixes |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Trie (prefix tree)** | Tree where each path from root to node represents a prefix of stored strings. |
| **Radix tree** | Compressed trie that collapses single-child chains into single nodes. |
| **Pre-computed top-K** | Storing the K most popular completions at each trie node during build time. |
| **Debounce** | Client-side delay (e.g., 200ms) before sending a request, avoiding per-keystroke queries. |
| **Exponential decay** | Weighting scheme where older events count less, controlled by a decay factor. |
| **Blue-green swap** | Loading a new trie alongside the old one, then atomically switching traffic. |

## Appendix C: References

- [57_search_engine](../57_search_engine/) -- Full search engine design
- [23_distributed_cache](../23_distributed_cache/) -- Caching suggestion responses
- [34_stream_processing](../34_stream_processing/) -- Real-time aggregation pipeline
- [04_cdn](../04_cdn/) -- Edge deployment of suggestion data
- [26_rate_limiter](../26_rate_limiter/) -- Rate limiting autocomplete API
