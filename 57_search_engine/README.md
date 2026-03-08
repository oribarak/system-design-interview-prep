# System Design Interview: Search Engine (Google Search) -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"We are designing a web-scale search engine that crawls the internet, indexes billions of web pages, and serves keyword queries with sub-second latency. The system must handle hundreds of thousands of queries per second, return highly relevant results ranked by a combination of content relevance and authority signals, and support features like autocomplete, spelling correction, and snippet generation. I will walk through the crawling pipeline, the inverted index architecture, the ranking system, and how we scale query serving across global data centers."

## 1) Clarifying Questions (2-5 min)

1. **Scope of crawling**: Are we designing the full crawl pipeline or focusing on the query-serving side? -- Assume both, but deep-dive on query serving.
2. **Content types**: Text web pages only, or also images, videos, news? -- Primarily text HTML pages.
3. **Freshness requirements**: How fresh must results be? -- News should be minutes-fresh; general pages can tolerate hours to days.
4. **Personalization**: Do we personalize results per user? -- Light personalization (location, language); no deep user-profile personalization initially.
5. **Autocomplete / Suggest**: Is query suggestion in scope? -- Yes, as a supporting feature.
6. **Ads**: Are we integrating sponsored results? -- Out of scope for this design.
7. **Scale**: How many pages indexed and QPS? -- 50 billion pages, 100K QPS peak.

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|---|---|
| Total pages indexed | 50 billion |
| Average page size (compressed text) | 50 KB |
| Raw corpus storage | 50B x 50 KB = 2.5 PB |
| Inverted index size (10% of corpus) | ~250 TB |
| QPS (peak) | 100,000 |
| Average query latency target | < 500 ms (p99 < 1s) |
| Average terms per query | 3-4 |
| Posting list reads per query | ~4 lists, merge/intersect |
| Daily new/updated pages crawled | ~5 billion |
| Crawl bandwidth | 5B x 50 KB / 86400s ~ 2.9 TB/s aggregate across crawlers |
| Query serving cluster size | 100K QPS / 1000 QPS per node ~ 100 query nodes per shard replica |

## 3) Requirements (3-5 min)

### Functional Requirements
- **Web Crawl**: Continuously discover, fetch, and store web pages.
- **Indexing**: Build and maintain an inverted index mapping terms to documents.
- **Query Processing**: Parse queries, retrieve matching documents, rank, and return top-K results with snippets.
- **Autocomplete**: Suggest query completions as the user types.
- **Spell Correction**: Detect and correct misspelled queries.
- **Snippet Generation**: Extract relevant text snippets from matching documents.

### Non-Functional Requirements
- **Latency**: p50 < 200 ms, p99 < 1 s for query serving.
- **Throughput**: 100K QPS sustained.
- **Availability**: 99.99% uptime for query serving.
- **Freshness**: Breaking news indexed within minutes; general pages within hours.
- **Relevance**: High precision and recall in top-10 results.
- **Scalability**: Horizontal scaling of both crawl pipeline and query serving.

## 4) API Design (2-4 min)

### Search Query
```
GET /search?q=<query>&page=<offset>&count=<num>&lang=<lang>&loc=<location>
Response: {
  results: [{ title, url, snippet, rank_score }],
  total_results: int,
  spelling_suggestion: string | null,
  related_queries: [string],
  page_info: { page, count, total_pages }
}
```

### Autocomplete
```
GET /suggest?prefix=<partial_query>&count=<num>
Response: { suggestions: [{ text, estimated_results }] }
```

### Crawl Submission (internal)
```
POST /crawl/submit
Body: { urls: [string], priority: int }
Response: { accepted: int, rejected: int }
```

## 5) Data Model (3-5 min)

### Document Store
| Field | Type | Notes |
|---|---|---|
| doc_id | uint64 | Globally unique document identifier |
| url | string | Canonical URL |
| title | string | Page title |
| content_hash | bytes | For deduplication |
| crawl_timestamp | timestamp | Last crawl time |
| page_rank | float | Authority score |
| language | string | Detected language |
| compressed_content | bytes | Stored in a columnar store or blob storage |

### Inverted Index
| Component | Description |
|---|---|
| Term Dictionary | Sorted list of all terms with pointers to posting lists |
| Posting List | For each term: list of (doc_id, term_frequency, positions[]) |
| Skip Pointers | Every N entries for fast intersection |

### Storage Choices
- **Document store**: Distributed blob storage (e.g., GFS/Colossus) for raw pages; Bigtable-style store for metadata.
- **Inverted index**: Custom SSTable-like format on SSD, memory-mapped for fast access.
- **Sharding**: Index sharded by doc_id range. Each shard holds a complete inverted index for its document subset. Query fanout to all shards.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary**: A crawl pipeline discovers and fetches pages into a distributed document store, a batch/streaming indexer builds sharded inverted indexes, and a query-serving layer fans out queries to index shards, merges ranked results, and returns them with generated snippets.

### Components

1. **URL Frontier / Scheduler** -- Manages the crawl queue with politeness rules and priority scheduling.
2. **Crawler Fleet** -- Distributed HTTP fetchers that download pages and push to the document store.
3. **Document Store** -- Stores raw page content, metadata, and content hashes for dedup.
4. **Indexer** -- Processes documents into inverted index segments; runs as batch MapReduce and near-real-time streaming jobs.
5. **Index Shards** -- Distributed index partitions, each serving a subset of documents.
6. **Query Parser** -- Tokenizes, stems, handles operators, spelling correction.
7. **Query Broker** -- Fans out parsed query to all index shards in parallel.
8. **Ranker** -- Scores documents using TF-IDF, BM25, PageRank, ML re-ranking.
9. **Snippet Generator** -- Extracts relevant text passages from stored documents.
10. **Autocomplete Service** -- Trie or prefix index over popular queries.
11. **Cache Layer** -- Query result cache (LRU) for popular queries.

## 7) Deep Dive #1: Inverted Index and Query Serving (8-12 min)

### Index Structure

The inverted index is the heart of the search engine. For each unique term across all documents, we maintain a **posting list**: an ordered sequence of document IDs where that term appears, along with per-document metadata (term frequency, field flags, position offsets).

**Term Dictionary**: A sorted array or trie of all unique terms (hundreds of millions). Each entry points to the start of its posting list on disk. The dictionary itself fits in memory (~10 GB for 500M terms at 20 bytes each).

**Posting List Encoding**: Posting lists are sorted by doc_id. We store gaps (delta encoding) and compress with variable-byte or PForDelta encoding. A posting list for a common term ("the") may have billions of entries; rare terms have short lists.

**Skip Pointers**: Every sqrt(N) entries, we store a skip pointer to allow fast intersection of posting lists without scanning every entry.

### Query Execution

1. **Parse**: Tokenize query "best Italian restaurants NYC" into terms: [best, italian, restaurant, nyc]. Apply stemming.
2. **Posting List Retrieval**: Fetch posting lists for each term from the index shard.
3. **Intersection / Union**: For AND semantics, intersect lists using skip pointers. For OR, union with score accumulation.
4. **Scoring**: For each candidate document, compute BM25 score incorporating term frequency, inverse document frequency, document length normalization.
5. **Top-K Selection**: Use a min-heap of size K to track top results.
6. **Shard Merge**: Query broker collects top-K from each shard, merges into global top-K.

### Sharding Strategy

- **Document-partitioned**: Each shard holds a random subset of documents and a complete inverted index for those documents. Every query must fan out to all shards.
- **Alternative -- Term-partitioned**: Each shard holds posting lists for a subset of terms. Multi-term queries require coordination. Less common due to hot-spot issues on popular terms.
- We use **document-partitioned** sharding. With 50B documents and 100M docs per shard, we need ~500 shards. Each shard is replicated 3x for availability and load balancing.

### Tiered Index

- **Tier 0 (Real-time)**: In-memory index for pages crawled in the last hour. Small, rebuilt frequently.
- **Tier 1 (Fast)**: High-quality pages (high PageRank). Searched first; can short-circuit if enough good results found.
- **Tier 2 (Full)**: Complete index. Searched only if Tier 1 results are insufficient.

## 8) Deep Dive #2: Ranking and Relevance (5-8 min)

### Multi-Stage Ranking Pipeline

1. **Stage 1 -- Retrieval (Index)**: BM25 scoring during posting list traversal. Returns top-1000 candidates per shard.
2. **Stage 2 -- Lightweight ML Ranker**: A fast linear model or small neural net re-ranks top-1000 candidates using features: BM25 score, PageRank, URL quality, freshness, domain authority. Runs in < 10 ms.
3. **Stage 3 -- Heavy ML Ranker**: A transformer-based model re-ranks top-100 candidates. Uses learned query-document embeddings, click-through rate predictions, BERT-style cross-attention. Runs in < 50 ms on GPU.
4. **Stage 4 -- Business Logic**: Apply diversity rules (no more than 2 results from the same domain), freshness boosts for time-sensitive queries, safe-search filtering.

### Key Ranking Signals

| Signal | Category | Description |
|---|---|---|
| BM25 | Textual | Term frequency with saturation and length normalization |
| PageRank | Authority | Link-based graph authority score |
| Anchor Text | Textual | Text of inbound links describes the page |
| Click-Through Rate | Behavioral | Historical click probability for query-URL pair |
| Dwell Time | Behavioral | How long users stay on the page after clicking |
| Freshness | Temporal | Recency of content update |
| Domain Authority | Authority | Aggregate quality score for the domain |
| Title/URL Match | Textual | Exact or partial match in title or URL |

### PageRank Computation

PageRank is computed offline via iterative MapReduce over the web graph. The web graph has ~50B nodes and ~500B edges. Each iteration is a matrix-vector multiplication. Typically converges in 50-100 iterations. The output is a static score per page, updated weekly.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Index partitioning | Document-partitioned | Term-partitioned | Simpler query execution; avoids hot spots on popular terms |
| Index format | Custom SSTable on SSD | Elasticsearch/Lucene | Full control over encoding, compression, skip pointers |
| Real-time indexing | Streaming + batch hybrid | Pure batch | Freshness for news; batch for efficiency on bulk corpus |
| Ranking | Multi-stage cascade | Single model | Latency budget requires cheap first pass, expensive re-rank on fewer docs |
| Snippet generation | On-the-fly from stored content | Pre-computed | Storage savings; snippets are query-dependent anyway |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (1M QPS, 500B pages)
- Increase shard count to ~5,000. Add more replicas per shard for read throughput.
- Expand cache layer; popular queries (Zipf distribution) mean cache hit rate > 60%.
- Move heavy ML ranker to dedicated GPU clusters with batched inference.
- Shard the autocomplete trie by prefix range.

### 100x (10M QPS, 5T pages)
- Multi-region deployment with geo-routing. Each region has a full index replica.
- Aggressive tiered indexing: only high-quality pages in Tier 1 (searched for 90% of queries).
- Pre-compute results for the top 10,000 queries (covers ~15% of traffic).
- Move to custom hardware (TPUs) for ranking inference.
- Hierarchical query broker: regional brokers aggregate from local shard clusters.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Index shard replication**: Each shard has 3 replicas across failure domains. Query broker routes to healthy replicas.
- **Graceful degradation**: If a shard is unavailable, return partial results from remaining shards with a quality indicator.
- **Crawl pipeline resilience**: URL frontier persists to disk. Crawler failures re-enqueue URLs. Idempotent processing.
- **Index build isolation**: New index versions are built offline and swapped in atomically (blue-green deployment of index segments).
- **Circuit breakers**: If heavy ML ranker is overloaded, fall back to lightweight ranker only.
- **Chaos testing**: Regularly kill random shard replicas to verify failover works.

## 12) Observability and Operations (2-4 min)

- **Query latency histograms**: p50, p95, p99 broken down by stage (retrieval, ranking, snippet gen).
- **Shard health dashboard**: Per-shard QPS, latency, error rate, replica status.
- **Index freshness metrics**: Age of newest document in each shard; lag between crawl and index availability.
- **Crawl metrics**: Pages/sec crawled, fetch error rates by domain, politeness delay compliance.
- **Relevance metrics**: Online A/B testing of ranking changes using click-through rate, abandonment rate, query reformulation rate.
- **Alerting**: Page on shard unavailability, latency spike > 2x baseline, crawl throughput drop > 30%.

## 13) Security (1-3 min)

- **Crawl safety**: Respect robots.txt. Rate-limit per domain to avoid being flagged as an attacker. Sanitize fetched content to prevent XSS in snippet display.
- **Query injection**: Sanitize query input; no raw query strings passed to backend systems.
- **DDoS protection**: Rate limiting per IP/API key at the edge. CDN-based absorption of volumetric attacks.
- **Safe search**: Content classification to filter explicit/harmful results.
- **Data privacy**: Do not log personally identifiable information in query logs used for ranking improvement.

## 14) Team and Operational Considerations (1-2 min)

- **Crawl team**: Owns URL frontier, fetcher fleet, document store. Focused on coverage and freshness.
- **Index team**: Owns indexer, index format, shard management. Focused on index build speed and compression.
- **Ranking team**: Owns query parser, ranking pipeline, ML models. Focused on relevance metrics.
- **Serving team**: Owns query broker, cache, load balancing. Focused on latency and availability.
- **On-call rotation**: Separate rotations for crawl pipeline (lower urgency) and query serving (high urgency).

## 15) Common Follow-up Questions

1. **How do you handle duplicate pages?** -- Content hashing (SimHash/MinHash) during crawl. Dedup at index time by keeping only the canonical URL.
2. **How does autocomplete work?** -- Trie built from popular query logs, updated hourly. Prefix lookup returns top completions ranked by frequency and recency.
3. **How do you handle multi-language search?** -- Language detection on crawled pages. Separate indexes per language. Query language inferred from user locale and query text.
4. **How do you prevent SEO spam?** -- Web spam classifier using link analysis (link farms detection), content quality signals, and manual penalties.
5. **How do you handle real-time events?** -- Streaming indexer with a dedicated real-time tier. Trending topic detection triggers aggressive re-crawl of relevant domains.

## 16) Closing Summary (30-60s)

"We designed a search engine with three main pipelines: a distributed crawler that continuously fetches pages into a document store, a hybrid batch/streaming indexer that builds document-partitioned inverted indexes across 500+ shards, and a query-serving layer that fans out queries, retrieves candidates via BM25, applies multi-stage ML ranking, and returns results with generated snippets -- all within 200 ms p50 latency at 100K QPS. The architecture scales horizontally by adding shards and replicas, uses tiered indexing for efficiency, and degrades gracefully when components fail."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Guidance |
|---|---|---|
| Posting list compression | PForDelta | VByte for smaller lists; PForDelta for large lists |
| BM25 k1 (term frequency saturation) | 1.2 | Higher for longer documents |
| BM25 b (length normalization) | 0.75 | Lower if document length varies widely |
| Top-K per shard | 1000 | Increase for recall; decrease for latency |
| Heavy ranker candidate count | 100 | Balance between relevance and GPU cost |
| Cache TTL | 5 minutes | Shorter for news-heavy queries |
| Crawl politeness delay | 1 req/sec/domain | Adjust per domain based on server capacity |
| Tier 1 index size | Top 1B pages | Larger tier = better results, more memory |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| Inverted Index | Data structure mapping terms to the list of documents containing them |
| Posting List | Sorted list of document IDs (and metadata) for a given term |
| BM25 | Probabilistic ranking function based on term frequency and document length |
| PageRank | Link-based algorithm that assigns authority scores to web pages |
| Skip Pointer | Index acceleration structure enabling fast list intersection |
| SimHash | Locality-sensitive hash for near-duplicate detection |
| URL Frontier | Priority queue managing which URLs to crawl next |
| TF-IDF | Term Frequency-Inverse Document Frequency; basic relevance scoring |
| Tiered Index | Multiple index levels searched in priority order |

## Appendix C: References

- Brin, S. and Page, L. "The Anatomy of a Large-Scale Hypertextual Web Search Engine" (1998)
- Manning, Raghavan, Schutze. "Introduction to Information Retrieval" (Cambridge University Press)
- Dean, J. "Challenges in Building Large-Scale Information Retrieval Systems" (WSDM 2009)
- Robertson, S. and Zaragoza, H. "The Probabilistic Relevance Framework: BM25 and Beyond" (2009)
- Google Research Blog: various posts on search infrastructure
