# System Design Interview: Google Maps / Location Service — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a mapping and navigation service like Google Maps that provides map rendering, place search, geocoding, route planning, real-time navigation with traffic, and ETA estimation. The core challenges are serving map tiles to billions of requests per day, computing shortest paths on a road graph with hundreds of millions of edges, incorporating real-time traffic data, and providing turn-by-turn navigation. I'll focus on the map tile serving system, the routing engine, and real-time traffic integration."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What features? Map display, directions, place search, street view? | Map display, directions (driving, walking, transit), place search, ETA |
| 2 | How many users? | 1B monthly active users, 50M daily direction requests |
| 3 | Global coverage? | Yes, worldwide road network |
| 4 | Real-time traffic? | Yes, from GPS traces of users + historical patterns |
| 5 | Multi-modal routing (car, walk, bike, transit)? | Yes, all four |
| 6 | Offline support? | Yes, downloadable map regions |
| 7 | How often is map data updated? | Road network: weekly; traffic: real-time (every 1-2 minutes) |
| 8 | Turn-by-turn navigation? | Yes, with re-routing on deviation |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Monthly active users | 1B |
| Daily map tile requests | 50B (each map view loads 10-20 tiles) |
| Tile request QPS | 50B / 86,400 ~ 580K QPS avg, ~1.5M QPS peak |
| Direction requests/day | 50M |
| Direction QPS | 580 avg, 1,700 peak |
| Road graph nodes | 500M intersections |
| Road graph edges | 1B road segments |
| Graph size in memory | ~50 GB (compressed adjacency list) |
| Map tile storage (all zoom levels) | ~100 TB (pre-rendered raster) or ~5 TB (vector tiles) |
| Traffic data points/sec | 10M GPS probes/min from user devices |
| Traffic segment updates/min | 50M road segments updated |
| Single route computation | 50-200ms with precomputation |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Map rendering**: serve map tiles at multiple zoom levels (0-20)
2. **Route planning**: compute optimal route between origin and destination
3. **ETA estimation**: predict travel time considering real-time traffic
4. **Place search**: find places by name or category near a location
5. **Geocoding/reverse geocoding**: convert between addresses and lat/lng
6. **Turn-by-turn navigation**: real-time guidance with re-routing on deviation
7. **Traffic layer**: display real-time traffic conditions on map

### Non-Functional Requirements
- **Tile latency**: < 100ms p99 for tile serving
- **Routing latency**: < 500ms for routes up to 500 km; < 2s for cross-country
- **Availability**: 99.99% for tile serving; 99.9% for routing
- **Freshness**: traffic data reflected within 2 minutes
- **Scale**: 1.5M QPS peak for tiles, 1,700 QPS for routing
- **Accuracy**: routes within 5% of optimal; ETA within 10% of actual

## 4) API Design (2-4 min)

### Map Tiles
```
GET /v1/tiles/{z}/{x}/{y}.pbf
// z: zoom level (0-20), x/y: tile coordinates
// Returns: vector tile (Protocol Buffers) or raster tile (PNG)
// Headers: Cache-Control, ETag for CDN caching
```

### Directions
```
POST /v1/directions
{
  "origin": {"lat": 37.7749, "lng": -122.4194},
  "destination": {"lat": 34.0522, "lng": -118.2437},
  "mode": "driving",
  "departure_time": "2024-03-07T16:00:00Z",
  "alternatives": true,
  "avoid": ["tolls", "highways"]
}
Response: {
  "routes": [{
    "distance_m": 613000,
    "duration_sec": 21600,
    "duration_in_traffic_sec": 25200,
    "polyline": "encoded_polyline_string",
    "steps": [
      {"instruction": "Head south on Market St", "distance_m": 500, "duration_sec": 60, "polyline": "..."}
    ]
  }]
}
```

### Place Search
```
GET /v1/places/search?query=coffee&lat=37.77&lng=-122.42&radius=1000
Response: { "results": [{"name": "Blue Bottle Coffee", "lat": 37.776, "lng": -122.418, "rating": 4.5, ...}] }
```

### Geocoding
```
GET /v1/geocode?address=1600+Amphitheatre+Parkway
GET /v1/reverse-geocode?lat=37.4220&lng=-122.0841
```

## 5) Data Model (3-5 min)

### Core Entities

**Road Graph** (in-memory)
| Field | Type | Notes |
|-------|------|-------|
| node_id | int64 | intersection ID |
| lat, lng | float | position |
| edges | list | adjacent segments |

**Road Segment (Edge)**
| Field | Type | Notes |
|-------|------|-------|
| edge_id | int64 | |
| from_node | int64 | |
| to_node | int64 | |
| distance_m | int | static |
| speed_limit_kmh | int | static |
| road_class | enum | highway, arterial, local |
| is_oneway | bool | |
| live_speed_kmh | float | real-time from traffic |
| historical_speed | float[] | by time-of-week (1,008 slots) |

**Map Tile**
| Field | Type | Notes |
|-------|------|-------|
| z, x, y | int | tile coordinates |
| data | bytes | vector or raster tile |
| version | int | map data version |
| updated_at | timestamp | |

**Place**
| Field | Type | Notes |
|-------|------|-------|
| place_id | UUID | PK |
| name | string | |
| lat, lng | float | |
| category | string | restaurant, gas station, etc. |
| address | string | |
| rating | float | |

### Storage Choices
- **Road graph**: custom binary format loaded into memory (50 GB compressed)
- **Map tiles**: S3 (origin) + CDN (CloudFront/Akamai) -- tiles are static, CDN-cacheable
- **Places**: Elasticsearch -- full-text search + geo queries
- **Traffic data**: Redis (live segment speeds) + time-series store (historical)
- **Precomputed routing shortcuts**: on-disk index (contraction hierarchies)

## 6) High-Level Architecture (5-8 min)

The system has three major subsystems: (1) map tile serving via CDN, (2) routing engine with precomputed graph shortcuts and real-time traffic overlay, and (3) traffic data pipeline that aggregates GPS probes into per-segment speed estimates.

### Components
- **Tile Server**: serves pre-rendered vector/raster tiles from S3 through CDN
- **Routing Engine**: computes shortest paths on road graph using contraction hierarchies
- **Traffic Pipeline**: ingests GPS probes, map-matches to road segments, computes live speeds
- **ETA Service**: combines route distance with live/predicted traffic speeds
- **Place Search Service**: Elasticsearch-powered place and POI search
- **Geocoding Service**: address-to-coordinates and reverse lookup
- **Navigation Service**: manages active navigation sessions, re-routes on deviation
- **Map Data Pipeline**: processes OSM/survey data into tiles and road graph (offline, weekly)

## 7) Deep Dive #1: Routing Engine (8-12 min)

### Why Not Dijkstra?
Plain Dijkstra on a 500M-node graph would explore millions of nodes per query, taking 5-10 seconds. We need precomputation.

### Contraction Hierarchies (CH)

**Precomputation (offline, takes hours):**
1. Order all nodes by "importance" (highways > local roads)
2. Iteratively contract the least important node:
   - For node v being contracted, check all pairs of neighbors (u, w)
   - If the shortest path u->w goes through v, add a shortcut edge u->w
   - Remove v from the active graph
3. Result: a hierarchy where important nodes (highway junctions) are at the top

**Query (online, < 50ms):**
1. Run bidirectional Dijkstra: forward from origin (only going "up" the hierarchy), backward from destination (only going "up")
2. Both searches meet at some high-level node
3. Unpack shortcut edges to recover the actual path
4. Explores only ~1,000-5,000 nodes instead of millions

**Performance:**
- Precomputation: 2-4 hours for continental graph
- Query: 1-10ms for continental routes (before traffic adjustment)
- Memory: ~60 GB for graph + shortcuts

### Integrating Real-Time Traffic

CH is precomputed with static edge weights. For live traffic:

**Approach: Customizable Contraction Hierarchies (CCH)**
1. Precompute the contraction order and shortcut structure once (topology doesn't change often)
2. At runtime, update edge weights with live traffic speeds
3. Recompute shortcut weights bottom-up (takes 1-2 seconds for the entire graph)
4. Run CH query with updated weights

**Traffic Weight Update Cycle:**
- Every 1-2 minutes, traffic pipeline publishes new segment speeds
- CCH customization runs: updates all shortcut weights (1-2 seconds)
- New queries use updated weights
- Old queries in flight use previous weights (eventual consistency)

### Alternative Routes
- Run CH query, find primary route
- Penalize edges on primary route (2x weight), re-run query for alternative
- Repeat once more for third alternative
- Deduplicate: alternatives must differ by > 20% of edges

### Multi-Modal Routing
- Separate graphs for driving, walking, cycling
- Transit routing: RAPTOR algorithm on GTFS schedule data
- Multi-modal: combine walking graph + transit graph with transfer nodes at stations

## 8) Deep Dive #2: Real-Time Traffic Pipeline (5-8 min)

### Data Sources
- **GPS probes**: 10M probes/minute from user devices (anonymized)
- **Historical patterns**: average speeds by road segment, time-of-week (168 hours * 6 10-min buckets = 1,008 slots)
- **Incidents**: road closures, accidents from partner data feeds
- **Events**: concerts, sports games affecting local traffic

### Map Matching Pipeline
GPS probes are noisy (10-20m accuracy) and must be matched to actual road segments:

1. **Ingest**: GPS probes arrive via Kafka (lat, lng, timestamp, speed, heading)
2. **Map matching**: Hidden Markov Model (HMM) matches sequence of GPS points to most likely road segments
   - Emission probability: distance from GPS point to road segment
   - Transition probability: routing distance between consecutive segments
3. **Speed computation**: for each road segment with matched probes:
   - Aggregate speeds from all probes in the last 5 minutes
   - Weighted average (more recent probes weighted higher)
   - Minimum 3 probes required; otherwise fall back to historical
4. **Publish**: updated segment speeds written to Redis, consumed by routing engine

### Traffic Flow Estimation
- **Coverage problem**: not every road segment has GPS probes
- **Solution**: propagate speeds from observed segments to adjacent unobserved segments using road class similarity and historical ratios
- **Confidence score**: segments with many probes have high confidence; extrapolated segments have lower confidence
- **Historical blend**: `live_speed = alpha * observed_speed + (1 - alpha) * historical_speed`, where alpha depends on probe count

### ETA Prediction
```
ETA = sum over route segments of (segment_length / predicted_speed(segment, time))
```
- Predicted speed accounts for time-of-arrival at each segment (not just current conditions)
- ML model trained on historical trips adjusts ETA for systematic biases (turning delays, traffic light waits)

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| Contraction Hierarchies | A* with landmarks | CH is 100-1000x faster for long routes; A* better for very short routes |
| Vector tiles | Raster tiles | Vector tiles are 80% smaller, render client-side with styling flexibility |
| CDN-cached tiles | Server-rendered on demand | Pre-rendered + CDN gives < 50ms latency globally; dynamic rendering can't compete |
| HMM map matching | Nearest-segment matching | HMM handles noise and ambiguity much better, especially at intersections |
| Redis for live traffic | In-memory in routing service | Redis decouples traffic pipeline from routing; multiple routing replicas share one source |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (5B daily direction requests)
- **Regional routing shards**: partition graph by continent/country, route queries to regional shard
- **Cross-region routing**: hierarchical approach -- local routing within regions, highway-only graph for inter-region
- **Tile serving**: add CDN PoPs, increase S3 replication
- **Traffic pipeline**: increase Kafka partitions, parallelize map matching

### 100x Scale (15M tile QPS, 50B direction requests/day)
- **Edge tile caching**: serve most popular tiles from edge servers without CDN origin hit
- **GPU-accelerated routing**: parallel shortest-path computation on GPU for batch queries
- **Predictive tile pre-fetching**: predict which tiles user will need next based on viewport movement
- **Federated routing**: each country operates independent routing infrastructure with border handoff
- **Compressed graph**: graph compression techniques reduce memory from 50 GB to 15 GB

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| CDN cache miss storm | Origin tile server behind auto-scaling group; deduplication of identical tile requests |
| Routing engine crash | Stateless replicas; graph loaded from shared storage at startup (2-3 min warmup) |
| Traffic pipeline lag | Fall back to historical traffic patterns; flag ETA as "estimated without live traffic" |
| GPS probe dropout | Historical speed patterns cover 100% of segments; live data is enhancement |
| Graph data corruption | Canary deployment of new graph data; rollback to previous version if routing quality drops |
| Cross-continent route failure | Route in segments: origin-to-border, border-to-border, border-to-destination |

### Tile Serving Resilience
- Tiles are immutable per version -- perfect for CDN caching (TTL: 7 days, cache hit rate > 99%)
- Multiple S3 regions for origin redundancy
- If primary CDN fails, fallback to secondary CDN provider

## 12) Observability and Operations (2-4 min)

### Key Metrics
- Tile serving latency p50/p99 (target: < 100ms)
- CDN cache hit rate (target: > 99%)
- Routing query latency p50/p99 (target: < 500ms)
- ETA accuracy (predicted vs actual, target: within 10%)
- Traffic pipeline freshness (lag from GPS probe to speed update)
- Map matching accuracy (% of probes successfully matched)
- Route quality (% of users who deviate from suggested route)

### Alerts
- CDN cache hit rate drops below 95% (possible cache invalidation issue)
- Routing p99 exceeds 2 seconds
- Traffic pipeline lag exceeds 5 minutes
- ETA error exceeds 20% averaged over 1 hour

## 13) Security (1-3 min)

- **Location privacy**: GPS probes are anonymized (no user IDs); aggregated at segment level
- **Differential privacy**: noise added to traffic data to prevent individual tracking
- **API key management**: per-developer API keys with usage quotas
- **Tile access control**: public tiles freely cached; premium features (traffic layer) require auth
- **Rate limiting**: per-API-key rate limits to prevent scraping of map data
- **Data licensing**: map data usage governed by OSM license or commercial license terms

## 14) Team and Operational Considerations (1-2 min)

- **Map data team (5-8 engineers)**: OSM processing, tile generation, map data pipeline
- **Routing team (5-8 engineers)**: contraction hierarchies, multi-modal routing, navigation
- **Traffic team (4-6 engineers)**: GPS ingestion, map matching, speed estimation, ETA models
- **Search/Geocoding team (3-5 engineers)**: place search, address parsing, geocoding
- **Infrastructure team (4-6 engineers)**: CDN management, tile serving, global deployment
- **On-call**: separate rotations for tile serving (high QPS) and routing (complex debugging)

## 15) Common Follow-up Questions

**Q: How does turn-by-turn navigation work?**
A: Client sends GPS position every second. Server compares position to planned route. If deviation > 50m, trigger re-route (new routing query from current position). Navigation instructions pre-computed along route; client triggers voice prompts based on distance to next maneuver.

**Q: How do you handle road closures or new roads?**
A: Road closures: update edge weights to infinity in the traffic layer (takes effect within 2 minutes). New roads: require graph rebuild (weekly cycle), but can be hot-patched for critical additions.

**Q: How does offline maps work?**
A: User downloads a region's vector tiles + routing graph subset. Routing runs locally using the same CH algorithm. No traffic data offline; uses historical patterns. Map data updates when reconnected.

**Q: How is geocoding implemented?**
A: Address parsing (NLP to extract street, city, etc.) + lookup in address database (inverted index by tokens). Fuzzy matching for typos. Ranked by relevance and proximity to user's location.

**Q: How do you handle routing across time zones or international borders?**
A: Graph is global but partitioned by region. Cross-border routes use stitching at border nodes. Time zone handling is in ETA calculation, not in routing. Some borders require special handling (restricted crossings).

## 16) Closing Summary (30-60s)

"I've designed a mapping and navigation system with three pillars: CDN-served vector tiles with 99%+ cache hit rate for sub-100ms map rendering, Contraction Hierarchies for sub-500ms routing on a 500M-node road graph, and a real-time traffic pipeline that map-matches 10M GPS probes per minute to compute live segment speeds. The routing engine integrates traffic via Customizable Contraction Hierarchies, updating edge weights every 1-2 minutes. The system scales through regional graph sharding, CDN distribution, and hierarchical routing for cross-region trips."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| tile_cache_ttl_days | 7 | 1 - 30 | CDN cache duration; longer = higher hit rate, staler tiles |
| traffic_aggregation_window_min | 5 | 2 - 15 | Window for averaging GPS probe speeds |
| min_probes_for_live_speed | 3 | 1 - 10 | Minimum probes before trusting live speed over historical |
| historical_blend_alpha | 0.7 | 0.3 - 1.0 | Weight of live data vs historical (1.0 = all live) |
| ch_customization_interval_sec | 90 | 30 - 300 | Frequency of updating CH shortcut weights with live traffic |
| max_routing_alternatives | 3 | 1 - 5 | Number of alternative routes to compute |
| reroute_deviation_threshold_m | 50 | 20 - 100 | Distance from route before triggering re-route |
| vector_tile_max_zoom | 20 | 16 - 22 | Maximum zoom level for pre-rendered tiles |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| Contraction Hierarchies (CH) | Routing algorithm that precomputes shortcuts for fast shortest-path queries |
| CCH | Customizable CH -- allows edge weight updates without full recomputation |
| Vector Tiles | Map tiles containing geometric data (not pixels); rendered client-side |
| Map Matching | Aligning noisy GPS points to road network segments |
| HMM | Hidden Markov Model, used for probabilistic map matching |
| Geohash | Spatial encoding for partitioning geographic space |
| RAPTOR | Algorithm for public transit routing on timetable data |
| GTFS | General Transit Feed Specification -- standard format for transit schedules |
| Polyline Encoding | Compact representation of a path as a string |
| Tile Pyramid | Hierarchy of map tiles at different zoom levels (z=0: 1 tile, z=20: billions) |

## Appendix C: References

- Google Maps technical overview (Google Research blog)
- "Contraction Hierarchies: Faster and Simpler Hierarchical Routing" (Geisberger et al.)
- OpenStreetMap data model and usage
- Mapbox Vector Tile specification
- "Fast Map Matching" (Newson & Krumm, 2009)
- Uber H3 hexagonal spatial index
- RAPTOR transit routing algorithm (Delling et al.)
- Protocol Buffers for vector tile encoding
