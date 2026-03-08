# System Design Interview: Google Analytics — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a web analytics platform like Google Analytics that collects, processes, and visualizes user interaction data from millions of websites. The system must handle a massive volume of event ingestion from client-side JavaScript trackers, process the events through a real-time and batch pipeline, store pre-aggregated metrics for fast dashboard queries, and support custom dimensions and segments. I will focus on the event collection infrastructure, the stream processing pipeline that sessionizes and aggregates events, the OLAP storage layer for interactive queries, and the real-time reporting path. The key challenges are handling billions of events per day at low cost, computing sessions from disconnected page views, and serving sub-second dashboard queries over large time ranges."

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we building a full Google Analytics clone, or the core tracking + reporting platform?
- Do we need real-time reporting (last 30 minutes) in addition to batch-processed historical data?
- Should we support custom events and dimensions beyond standard page views?

**Scale**
- How many tracked websites/apps? Assume 5 million properties.
- What is the event ingestion rate? Assume 2 million events per second (peak).
- How many dashboard queries per second? Assume 50,000 queries/sec.

**Policy / Constraints**
- What is the acceptable delay for data to appear in reports? Assume < 5 minutes for real-time view, < 4 hours for full processed reports.
- Do we need to support raw event export, or only pre-aggregated reports?
- What privacy requirements exist (GDPR, cookie consent, data retention)?

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Tracked properties | 5,000,000 websites/apps |
| Peak event ingestion | 2,000,000 events/sec |
| Average event size | 500 bytes (URL, referrer, user agent, dimensions) |
| Ingestion bandwidth | 2M x 500B = 1 GB/s |
| Daily events | 2M x 86,400 (avg ~1M/s) = ~86 billion/day |
| Daily raw storage | 86B x 500B = ~43 TB/day raw |
| Compressed (5:1) | ~8.5 TB/day |
| 90-day raw retention | ~765 TB |
| Pre-aggregated storage | ~10% of raw = ~77 TB |
| Dashboard queries | 50,000/sec |
| Unique users tracked daily | ~2 billion |

## 3) Requirements (3-5 min)

### Functional
- Collect page view events, custom events, and e-commerce events from a JavaScript tracker embedded in websites.
- Identify and sessionize user interactions (group page views into sessions with 30-minute inactivity timeout).
- Compute standard metrics: page views, sessions, users, bounce rate, session duration, conversion rate.
- Support dimensional breakdowns: by page, referrer, geography, device, browser, custom dimensions.
- Provide real-time reporting (events in the last 30 minutes) and historical reporting (up to 90 days).
- Allow audience segmentation and funnel analysis.

### Non-functional
- Ingestion: handle 2M events/sec with < 100 ms collection latency.
- Real-time pipeline: events visible in real-time reports within 5 minutes.
- Batch pipeline: full processed reports available within 4 hours.
- Query latency: dashboard queries return in < 2 seconds for standard reports.
- Cost efficiency: storage and compute costs scale sub-linearly with event volume through aggregation.
- Privacy: support user opt-out, data deletion requests, and configurable retention.

## 4) API Design (2-4 min)

```
GET /collect?v=1&tid=UA-123&cid=abc&t=pageview&dl=https://example.com/page&dr=https://google.com&dt=Page%20Title&ul=en-us&sr=1920x1080
  (Tracking pixel / beacon endpoint - returns 1x1 GIF or 204)
  Also accepts POST with JSON body for batch event submission.

POST /collect/batch
  Body: {events: [{tid: "UA-123", cid: "abc", t: "event", ec: "Video", ea: "Play", ...}, ...]}
  Returns: 204 No Content

GET /api/v1/reports
  Params: property_id, start_date, end_date, metrics (pageviews, sessions, users),
          dimensions (page, country, device), filters, segment, sort, limit
  Returns: {rows: [{page: "/home", pageviews: 50000, sessions: 32000, bounce_rate: 0.45}, ...],
            totals: {pageviews: 1200000, sessions: 800000}, sampled: false}

GET /api/v1/realtime
  Params: property_id, metrics (active_users, pageviews_per_minute)
  Returns: {active_users: 4523, top_pages: [{page: "/sale", active: 892}, ...]}

GET /api/v1/funnel
  Params: property_id, steps [{page: "/cart"}, {page: "/checkout"}, {page: "/thankyou"}], date_range
  Returns: {steps: [{name: "/cart", users: 10000}, {name: "/checkout", users: 4200}, {name: "/thankyou", users: 3100}]}
```

## 5) Data Model (3-5 min)

### Raw Event (Kafka / Object Storage)

| Field | Type | Description |
|-------|------|-------------|
| event_id | UUID | Unique event identifier |
| property_id | string | Website/app identifier (e.g., UA-123) |
| client_id | string | Anonymous user identifier (cookie-based) |
| session_id | string | Computed during sessionization |
| event_type | enum | pageview, event, transaction, timing |
| timestamp | int64 | Event timestamp (milliseconds) |
| page_url | string | Full page URL |
| page_title | string | Page title |
| referrer | string | Referring URL |
| country | string | Geo-resolved from IP |
| city | string | Geo-resolved from IP |
| device_type | enum | desktop, mobile, tablet |
| browser | string | User agent parsed |
| os | string | Operating system |
| screen_resolution | string | e.g., "1920x1080" |
| language | string | Browser language |
| custom_dimensions | map<string,string> | User-defined dimensions |
| event_category | string | For custom events |
| event_action | string | For custom events |
| event_label | string | For custom events |
| event_value | float | For custom events |

### Pre-Aggregated Report Table (OLAP Store / ClickHouse)

| Field | Type | Description |
|-------|------|-------------|
| property_id | string | Partition key |
| date | date | Aggregation date |
| dimension_key | string | Combination of dimensions (page+country+device) |
| dimension_values | map<string,string> | {page: "/home", country: "US", device: "mobile"} |
| pageviews | uint64 | Count of page views |
| sessions | uint64 | Count of unique sessions |
| users | uint64 | Count of unique client_ids (HyperLogLog) |
| bounces | uint64 | Sessions with only one page view |
| total_session_duration | uint64 | Sum of session durations in seconds |
| conversions | uint64 | Count of conversion events |

**Storage choices**:
- Raw events: Kafka for ingestion, Parquet files on S3 for long-term storage.
- Pre-aggregated: ClickHouse (columnar OLAP) for fast analytical queries, partitioned by (property_id, date).
- Real-time: Redis or Apache Druid for the real-time view (last 30 minutes).

## 6) High-Level Architecture (5-8 min)

**Dataflow**: The JavaScript tracker fires events to a collection endpoint at the edge (CDN). Edge servers validate events, enrich with geo data, and publish to Kafka. A stream processor (Flink) consumes from Kafka, sessionizes events, computes real-time aggregates, and writes to the real-time store. A batch pipeline periodically reads from Kafka/S3, performs full sessionization and aggregation, and writes to the OLAP store. Dashboard queries hit the OLAP store for historical data and the real-time store for live data.

**Components**:
- **JavaScript Tracker**: client-side SDK that captures page views, events, and user properties. Sends beacons to the collection endpoint.
- **Collection Endpoint (Edge)**: CDN-hosted endpoint that accepts events, validates payloads, resolves geo from IP, and publishes to Kafka. Returns immediately (fire-and-forget from the client's perspective).
- **Kafka**: event bus with topic per property_id range. Provides durability and decouples collection from processing.
- **Stream Processor (Flink)**: consumes from Kafka, performs real-time sessionization, computes rolling aggregates, and writes to the real-time store and raw event archive.
- **Real-Time Store (Druid/Redis)**: holds the last 30 minutes of aggregated data for the real-time dashboard.
- **Batch Pipeline (Spark)**: runs every 1-4 hours, performs full sessionization with late-arriving events, computes all dimensions, and writes to the OLAP store.
- **OLAP Store (ClickHouse)**: columnar database for fast analytical queries on pre-aggregated data. Partitioned by property_id and date.
- **Query Service**: translates dashboard queries into SQL, executes against ClickHouse and the real-time store, merges results.
- **Raw Event Archive (S3 + Parquet)**: compressed raw events for ad-hoc queries, data export, and reprocessing.

**One-sentence summary**: A lambda-architecture analytics platform that collects events at the edge, processes them through both a real-time stream (Flink) and batch (Spark) pipeline, stores pre-aggregated metrics in a columnar OLAP database, and serves dashboards with sub-second query latency.

## 7) Deep Dive #1: Sessionization (8-12 min)

### What is Sessionization?

A session is a group of user interactions within a time window. The standard definition: a session ends after 30 minutes of inactivity or at midnight (to align with daily reporting).

Given: a sequence of events from client_id=abc, sorted by timestamp:
```
10:00 - pageview /home
10:02 - pageview /products
10:45 - pageview /home       (gap > 30 min, new session)
10:47 - pageview /checkout
```
This produces 2 sessions: {/home, /products} and {/home, /checkout}.

### Stream Sessionization (Real-Time)

Using Flink's session windows:

1. **Key by (property_id, client_id)**: all events for a user within a property are processed together.
2. **Session window with 30-minute gap**: Flink merges events into a session window. If no event arrives for 30 minutes, the window closes.
3. **On window close**: emit session metrics (page count, duration, entry page, exit page, bounce flag).

**Challenge**: late-arriving events. An event might arrive after the session window has closed (network delay, offline-to-online sync). Solutions:
- **Allowed lateness**: configure Flink to keep windows open for an additional 5 minutes past the gap.
- **Session merging**: if a late event falls within 30 minutes of an existing session, merge them.
- **Batch correction**: the batch pipeline re-sessionizes with complete data, correcting any real-time inaccuracies.

### Batch Sessionization (Correctness)

The batch pipeline (Spark) runs every 1-4 hours on raw events from S3:

1. Read all events for a time range (e.g., last 4 hours + 1 hour buffer for late events).
2. Sort by (property_id, client_id, timestamp).
3. Walk through events: if the gap between consecutive events > 30 minutes, start a new session.
4. Assign session_id = hash(property_id, client_id, session_start_time).
5. Compute session-level metrics and aggregate by dimensions.
6. Write results to ClickHouse, replacing any previous real-time estimates.

### User Counting (HyperLogLog)

Counting unique users across billions of events is memory-intensive if done exactly. Instead, use HyperLogLog (HLL):
- Insert each client_id into an HLL sketch per (property_id, date, dimension_key).
- HLL provides an estimated count with ~2% error using only 12 KB of memory per sketch.
- HLL sketches are mergeable: to get unique users across multiple dates, merge the daily sketches.

This allows us to answer "how many unique users visited in the last 30 days?" without storing all 30 days of client_ids.

## 8) Deep Dive #2: OLAP Query Engine and Pre-Aggregation (5-8 min)

### Why Pre-Aggregation?

A property with 10 million page views per day cannot afford to scan all raw events for every dashboard query. Instead, pre-compute aggregates at multiple dimension granularities:

- **Hourly rollups**: pageviews, sessions, users by (property_id, hour, page, country, device).
- **Daily rollups**: same dimensions aggregated to daily.
- **Weekly/monthly rollups**: for long-range queries.

A typical dashboard query ("pageviews by page for last 7 days") reads 7 rows from the daily rollup instead of scanning 70 million raw events.

### ClickHouse Schema Optimization

ClickHouse stores data in columnar format, enabling:
- **Column pruning**: if the query only needs pageviews and sessions, only those columns are read from disk.
- **Compression**: integer columns compress extremely well (delta encoding, LZ4). 10:1 compression is typical.
- **Partitioning**: data is partitioned by (property_id, date). Queries always include these in WHERE clauses, so partitions are pruned efficiently.
- **Materialized views**: ClickHouse can maintain pre-aggregated materialized views that update automatically on insert.

### Handling Custom Dimensions

Users define custom dimensions (e.g., "logged_in_status", "experiment_variant"). These cannot be pre-aggregated for all combinations (combinatorial explosion). Approaches:

1. **Top N dimensions pre-aggregated**: for the most common custom dimensions, include them in the rollup tables.
2. **On-the-fly aggregation**: for rare custom dimension queries, scan the raw event data in Parquet on S3. Return results with a "sampled" flag if the data volume is too large.
3. **Sampling**: for properties with very high event volumes, sample events (e.g., 10% sample) and extrapolate. Google Analytics uses this approach for complex queries on large properties.

### Query Routing

The Query Service decides where to execute each query:
- Standard metrics + standard dimensions + date range < 90 days: ClickHouse pre-aggregated tables. Sub-second response.
- Custom dimensions or complex segments: ClickHouse raw event table (or Parquet on S3 via Trino/Presto). 5-30 second response.
- Real-time (last 30 minutes): Druid/Redis real-time store.
- Cross-property queries (rare, admin only): federated query across multiple ClickHouse shards.

## 9) Trade-offs and Alternatives (3-5 min)

### Lambda Architecture (Our Design) vs. Kappa Architecture
- **Lambda**: separate batch and stream pipelines. Batch is the source of truth, stream provides low-latency approximations. More complex (two code paths) but more accurate.
- **Kappa**: stream-only. Replay Kafka log for corrections. Simpler, but harder to handle complex sessionization and late-arriving events at scale.

### Pre-Aggregation vs. Raw Scans
- **Pre-aggregation**: sub-second queries, lower storage for common queries. But combinatorial explosion for many dimensions, and pre-agg cannot answer ad-hoc queries.
- **Raw scans**: flexible, can answer any query. But slow (minutes) for large datasets.
- **Our hybrid**: pre-aggregate common queries, fall back to raw scans for custom analyses.

### Cookie-Based vs. Cookieless Tracking
- **Cookie-based (traditional)**: reliable user identification via first-party cookies. Increasingly blocked by browsers.
- **Cookieless**: fingerprinting (privacy-invasive, unreliable) or server-side event collection (more privacy-friendly but less coverage).
- **Our approach**: first-party cookies with consent, server-side collection API as fallback, and privacy-preserving aggregation (no individual-level data in reports).

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (20M events/sec)
- **Edge collection scaling**: CDN handles collection at edge PoPs. No central bottleneck for ingestion.
- **Kafka scaling**: increase partitions to 10,000 across 200 brokers.
- **Flink scaling**: add more Flink task managers. Sessionization is keyed by (property_id, client_id), so it scales linearly with parallelism.
- **ClickHouse scaling**: shard by property_id hash across 100+ ClickHouse nodes. Each shard handles queries for its property subset.

### At 100x (200M events/sec)
- **Sampling at ingestion**: for very large properties (>1M events/day), sample events at the edge (e.g., 10%) and extrapolate in reports. Google Analytics does this.
- **Tiered processing**: Tier 1 (top 1% of properties by volume) gets dedicated processing pipelines. Tier 2 (rest) shares infrastructure.
- **Approximate algorithms everywhere**: HyperLogLog for users, Count-Min Sketch for top-N pages, t-digest for session duration percentiles.
- **Cold storage queries**: queries on data older than 30 days are served from Parquet on S3 via a query engine (Trino), with 10-60 second latency accepted.

## 11) Reliability and Fault Tolerance (3-5 min)

- **Collection endpoint**: stateless, CDN-hosted. If one PoP fails, clients are routed to the next nearest PoP. Events are fire-and-forget from the client (no retry by the tracker).
- **Kafka durability**: replication factor 3, min.insync.replicas=2. No event loss after Kafka acknowledgment.
- **Stream processing (Flink)**: checkpointing every 30 seconds. On failure, Flink restores from the last checkpoint and replays from Kafka.
- **Batch pipeline**: idempotent writes to ClickHouse. If a Spark job fails, it is retried from the beginning. Output overwrites previous results for the same time range.
- **ClickHouse HA**: ReplicatedMergeTree engine with ZooKeeper for replication. Survive single-node failures per shard.
- **Data loss budget**: accept up to 0.01% event loss at the collection layer (UDP beacons, network failures). This is acceptable for analytics (not financial data).

## 12) Observability and Operations (2-4 min)

- **Metrics**: events/sec ingested, Kafka consumer lag per topic, Flink checkpoint duration, ClickHouse query latency (p50/p99), batch pipeline duration, storage growth rate.
- **Dashboards**: per-property event volume, processing pipeline health, query performance by type, storage utilization by tier.
- **Alerts**: Kafka consumer lag > 5 minutes, Flink checkpoint failures, ClickHouse query timeout rate > 1%, batch pipeline not completed within 4-hour SLA.
- **Data quality**: monitor event validation failure rate, session duration distribution (anomaly detection), unique user count trends.

## 13) Security (1-3 min)

- **Privacy**: no PII in raw events (hash IP addresses, do not store raw IPs after geo-resolution). Support GDPR data deletion requests by marking client_ids for deletion and purging in the next batch run.
- **Consent management**: the JavaScript tracker respects consent signals (e.g., TCF). If consent is not given, do not set cookies and do not collect data.
- **API security**: dashboard API requires authentication (OAuth2). Property-level access control ensures users only see their own data.
- **Data isolation**: multi-tenant data in ClickHouse is isolated by property_id. Queries always include property_id in WHERE clause, enforced at the query service layer.

## 14) Team and Operational Considerations (1-2 min)

- **Teams**: Tracker SDK team (JavaScript, mobile), Collection/Ingestion team (edge, Kafka), Processing team (Flink, Spark, sessionization), Storage/Query team (ClickHouse, query optimization), Product team (dashboards, reports, UX).
- **Deployment**: tracker SDK version updates are gradual (CDN-hosted, cache-busted by version). Pipeline changes are deployed with dual-run validation (new + old pipeline, compare outputs).
- **Cost management**: the largest cost drivers are storage (S3 for raw events) and compute (Flink/Spark for processing). Aggressive rollup and retention policies are essential.

## 15) Common Follow-up Questions

1. **How do you handle ad blockers that block the tracker?** Use first-party collection endpoints (e.g., analytics.example.com proxied to the collection service). Server-side tracking via the Measurement Protocol for critical events.
2. **How do you compute bounce rate?** A bounce is a session with exactly one page view. Count single-page-view sessions / total sessions. Computed during sessionization.
3. **How do you attribute conversions to channels?** Store the referrer/campaign source on each event. During sessionization, attribute the session to its traffic source. For multi-touch attribution, track all touchpoints and apply a model (last-click, linear, time-decay).
4. **How do you handle cross-device tracking?** If the user logs in on multiple devices, a user_id (not client_id) links the sessions. Requires explicit user identification via the tracker API.
5. **How do you support real-time active user count?** Use a sliding HyperLogLog over the last 5 minutes, updated by the stream processor. Accurate within ~2%.

## 16) Closing Summary (30-60s)

> "We designed a web analytics platform that collects events via a JavaScript tracker and edge collection endpoints, processing 2 million events per second. Events flow through Kafka into a dual pipeline: a Flink stream processor for real-time sessionization and aggregation, and a Spark batch pipeline for complete historical processing. Pre-aggregated metrics are stored in ClickHouse for sub-second dashboard queries, with raw events archived in Parquet on S3 for ad-hoc analysis. Sessionization uses 30-minute gap detection, unique users are counted with HyperLogLog sketches, and the system scales through edge collection, Kafka partitioning, and ClickHouse sharding. The architecture balances real-time freshness with batch accuracy through the lambda pattern."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Effect |
|------|---------|--------|
| `session_timeout` | 30 min | Inactivity gap before a new session starts |
| `late_event_tolerance` | 5 min | How long to wait for late events in stream sessionization |
| `batch_interval` | 4 hours | How often the batch pipeline runs |
| `sampling_threshold` | 10M events/day | Properties above this threshold are sampled |
| `sampling_rate` | 10% | Fraction of events retained for large properties |
| `hll_precision` | 14 | HyperLogLog precision (higher = more accurate, more memory) |
| `rollup_dimensions` | [page, country, device, source] | Dimensions pre-aggregated in rollup tables |
| `retention_raw` | 90 days | How long raw events are kept |
| `retention_aggregated` | 3 years | How long pre-aggregated data is kept |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|-----------|
| **Sessionization** | Grouping individual events into sessions based on time gaps (typically 30-minute inactivity). |
| **HyperLogLog** | Probabilistic data structure for estimating unique counts with ~2% error and fixed memory. |
| **Bounce Rate** | Percentage of sessions with only a single page view. |
| **Pre-Aggregation** | Computing and storing aggregate metrics ahead of query time for fast dashboard performance. |
| **Lambda Architecture** | Processing pattern with both a real-time stream path and a batch path; batch corrects stream approximations. |
| **Tracking Pixel** | A 1x1 transparent image request used to collect analytics data from web pages. |
| **Dimension** | A categorical attribute used for grouping metrics (e.g., page, country, device type). |
| **Metric** | A quantitative measurement (e.g., pageviews, sessions, bounce rate). |

## Appendix C: References

- Google Analytics Measurement Protocol documentation
- ClickHouse documentation on MergeTree and materialized views
- Apache Flink session window documentation
- "The Lambda Architecture" — Nathan Marz
- HyperLogLog paper: Flajolet et al.
- Snowplow Analytics architecture (open-source alternative)
