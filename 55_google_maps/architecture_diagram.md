# Google Maps / Location Service — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients
        MOBILE[Mobile App<br/>Map rendering, Navigation]
        WEB[Web App<br/>Map display, Directions]
        API_CLIENT[Third-Party API Users]
    end

    subgraph CDN_Layer[CDN Layer]
        CDN[CDN<br/>CloudFront / Akamai<br/>99%+ cache hit rate]
    end

    subgraph Tile_System[Map Tile System]
        TILE_SVC[Tile Server<br/>Origin for CDN]
        TILE_STORE[(S3<br/>Vector Tiles ~5 TB)]
        TILE_GEN[Tile Generator<br/>Weekly batch from OSM]
    end

    subgraph Routing_System[Routing System]
        ROUTE_API[Routing API]
        ROUTE_ENGINE[Routing Engine<br/>Contraction Hierarchies]
        GRAPH[(Road Graph<br/>500M nodes, 1B edges<br/>~50 GB in memory)]
        CCH[CCH Customizer<br/>Update weights every 90s]
    end

    subgraph Traffic_System[Traffic Pipeline]
        GPS_INGEST[GPS Probe Ingestion<br/>10M probes/min]
        KAFKA_GPS[Kafka<br/>gps-probes topic]
        MAP_MATCH[Map Matching Service<br/>HMM-based]
        SPEED_AGG[Speed Aggregator<br/>5-min sliding window]
        TRAFFIC_STORE[(Redis<br/>Live segment speeds)]
        HIST_STORE[(Time-Series DB<br/>Historical patterns)]
    end

    subgraph Search_System[Place Search]
        SEARCH_API[Search API]
        ES[(Elasticsearch<br/>Places + POI)]
        GEOCODE[Geocoding Service]
    end

    subgraph Navigation[Navigation Service]
        NAV_SVC[Navigation Manager<br/>Active sessions]
        REROUTE[Re-routing Engine<br/>Deviation detection]
    end

    MOBILE -->|map tiles| CDN
    WEB -->|map tiles| CDN
    CDN -->|cache miss| TILE_SVC
    TILE_SVC --> TILE_STORE
    TILE_GEN -->|weekly| TILE_STORE

    MOBILE -->|directions| ROUTE_API
    WEB -->|directions| ROUTE_API
    API_CLIENT --> ROUTE_API
    ROUTE_API --> ROUTE_ENGINE
    ROUTE_ENGINE --> GRAPH
    CCH -->|update weights| GRAPH
    TRAFFIC_STORE -->|live speeds| CCH

    MOBILE -->|GPS probes| GPS_INGEST
    GPS_INGEST --> KAFKA_GPS
    KAFKA_GPS --> MAP_MATCH
    MAP_MATCH --> SPEED_AGG
    SPEED_AGG --> TRAFFIC_STORE
    SPEED_AGG --> HIST_STORE

    MOBILE -->|search| SEARCH_API
    SEARCH_API --> ES
    SEARCH_API --> GEOCODE

    MOBILE -->|navigation| NAV_SVC
    NAV_SVC --> REROUTE
    REROUTE --> ROUTE_ENGINE
```

## 2. Deep-Dive: Routing Engine with Contraction Hierarchies

```mermaid
flowchart TB
    subgraph Preprocessing[Offline Preprocessing - Hours]
        RAW_GRAPH[Raw Road Graph<br/>500M nodes, 1B edges]
        NODE_ORDER[Node Ordering<br/>Rank by importance<br/>Highway > Arterial > Local]
        CONTRACT[Node Contraction<br/>Remove low-importance nodes<br/>Add shortcut edges]
        CH_GRAPH[CH Graph<br/>Original + shortcut edges<br/>Hierarchical levels]
    end

    subgraph Live_Update[Live Weight Update - Every 90s]
        TRAFFIC_IN[Live Segment Speeds<br/>From Redis]
        HIST_IN[Historical Speeds<br/>For uncovered segments]
        BLEND[Speed Blending<br/>alpha * live + (1-alpha) * historical]
        BOTTOM_UP[Bottom-Up Weight Update<br/>Recompute shortcut weights<br/>1-2 seconds]
        UPDATED_CH[Updated CH Graph<br/>Ready for queries]
    end

    subgraph Query[Online Query - < 50ms]
        ORIGIN[Origin Node]
        DEST[Destination Node]
        FWD_SEARCH[Forward Dijkstra<br/>Only go UP hierarchy]
        BWD_SEARCH[Backward Dijkstra<br/>Only go UP hierarchy]
        MEETING[Meeting Point<br/>Top of hierarchy]
        UNPACK[Unpack Shortcuts<br/>Recover actual path]
        ROUTE[Full Route<br/>Turn-by-turn steps]
    end

    subgraph Alternatives[Alternative Routes]
        PRIMARY[Primary Route Found]
        PENALIZE[Penalize Primary Edges<br/>2x weight]
        REQUERY[Re-run CH Query]
        ALT_ROUTE[Alternative Route]
        DEDUP[Deduplicate<br/>Must differ by > 20%]
    end

    RAW_GRAPH --> NODE_ORDER
    NODE_ORDER --> CONTRACT
    CONTRACT --> CH_GRAPH

    TRAFFIC_IN --> BLEND
    HIST_IN --> BLEND
    BLEND --> BOTTOM_UP
    CH_GRAPH --> BOTTOM_UP
    BOTTOM_UP --> UPDATED_CH

    ORIGIN --> FWD_SEARCH
    DEST --> BWD_SEARCH
    UPDATED_CH --> FWD_SEARCH
    UPDATED_CH --> BWD_SEARCH
    FWD_SEARCH --> MEETING
    BWD_SEARCH --> MEETING
    MEETING --> UNPACK
    UNPACK --> ROUTE

    ROUTE --> PRIMARY
    PRIMARY --> PENALIZE
    PENALIZE --> REQUERY
    REQUERY --> ALT_ROUTE
    ALT_ROUTE --> DEDUP
```

## 3. Critical Path Sequence: Direction Request with Live Traffic

```mermaid
sequenceDiagram
    participant User as User (Mobile App)
    participant API as Routing API
    participant Geo as Geocoding Service
    participant Engine as CH Routing Engine
    participant Graph as Road Graph (Memory)
    participant Traffic as Traffic Store (Redis)
    participant ETA as ETA Service
    participant Hist as Historical Speed DB

    User->>API: POST /directions {origin: "123 Main St", dest: "456 Oak Ave", mode: driving}

    par Geocode both addresses
        API->>Geo: Geocode "123 Main St"
        Geo-->>API: {lat: 37.774, lng: -122.419}
        API->>Geo: Geocode "456 Oak Ave"
        Geo-->>API: {lat: 37.338, lng: -121.886}
    end

    API->>Engine: Route(origin_node, dest_node)

    Note over Engine,Graph: Bidirectional CH Search

    Engine->>Graph: Forward search from origin (up hierarchy)
    Engine->>Graph: Backward search from dest (up hierarchy)

    Note over Engine: Searches meet at highway junction node<br/>~2,000 nodes explored (vs millions for Dijkstra)

    Engine->>Engine: Unpack shortcut edges to full path
    Engine-->>API: Primary route: 380 edges, 65 km

    par Compute alternatives
        Engine->>Engine: Penalize primary edges (2x)
        Engine->>Graph: Re-run CH search
        Engine-->>API: Alternative route: 72 km
    end

    API->>ETA: Compute ETA for routes

    par Get traffic for route segments
        ETA->>Traffic: GET speeds for 380 segments
        Traffic-->>ETA: Live speeds (340 segments covered)
        ETA->>Hist: GET historical speeds (40 uncovered segments)
        Hist-->>ETA: Historical speeds for current time-of-week
    end

    ETA->>ETA: Sum segment_length / predicted_speed for each segment
    ETA->>ETA: Apply ML adjustment (turning delays, signals)
    ETA-->>API: Primary: 52 min (with traffic), Alt: 58 min

    API-->>User: {routes: [{distance: 65km, duration: 52min, steps: [...], polyline: "..."},<br/>{distance: 72km, duration: 58min, ...}]}

    Note over User,API: User starts navigation

    loop Every 1 second during navigation
        User->>API: GPS position update
        API->>API: Compare position to planned route
        alt Deviation > 50m
            API->>Engine: Re-route from current position
            Engine-->>API: New route
            API-->>User: Updated route + instructions
        else On route
            API-->>User: Next instruction + updated ETA
        end
    end
```
