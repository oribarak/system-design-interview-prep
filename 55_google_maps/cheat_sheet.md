# Google Maps / Location Service — Cheat Sheet

## Key Numbers
- 1B MAU, 580K tile QPS (1.5M peak), 580 routing QPS (1,700 peak)
- Road graph: 500M nodes, 1B edges, ~50 GB in memory
- Map tiles: ~5 TB vector tiles, CDN cache hit rate > 99%
- 10M GPS probes/min for live traffic
- Route computation: < 50ms with CH (vs 5-10s for naive Dijkstra)
- Traffic update cycle: every 90 seconds
- ETA accuracy target: within 10% of actual

## Core Components
- **Tile Server + CDN**: pre-rendered vector tiles served via CDN with 99%+ hit rate, < 100ms
- **Routing Engine**: Contraction Hierarchies (CH) on road graph, explores ~2K nodes vs millions
- **CCH Customizer**: updates CH shortcut weights with live traffic every 90 seconds (1-2s recompute)
- **Traffic Pipeline**: GPS probes -> Kafka -> HMM map matching -> speed aggregation -> Redis
- **ETA Service**: sum(segment_length / predicted_speed) with ML adjustment for turns and signals
- **Place Search**: Elasticsearch with geo queries for POI search and geocoding

## Architecture in One Sentence
CDN-cached vector tiles serve map rendering at 1.5M QPS, Contraction Hierarchies route on a 500M-node graph in under 50ms, and a traffic pipeline map-matches 10M GPS probes/min to update edge weights every 90 seconds.

## Top 3 Trade-offs
1. **Contraction Hierarchies vs A***: CH is 100-1000x faster for long routes but requires hours of preprocessing
2. **Vector tiles vs raster tiles**: vectors are 80% smaller and style-flexible but require client-side rendering
3. **Live traffic blend vs pure live**: blending with historical handles coverage gaps but is less responsive to sudden changes

## Scaling Story
- 10x: regional graph sharding, more CDN PoPs, parallelized map matching
- 100x: hierarchical cross-region routing, GPU-accelerated batch routing, predictive tile pre-fetching

## Closing Statement
"CDN-served vector tiles at 99%+ cache hit rate. Contraction Hierarchies route in < 50ms by exploring ~2K nodes instead of millions. CCH customization integrates live traffic every 90 seconds. HMM map matching converts 10M GPS probes/min into per-segment speeds. ETA prediction within 10% accuracy using live + historical blend with ML adjustment."
