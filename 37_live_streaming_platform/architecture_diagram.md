# Live Streaming Platform -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Broadcaster["Broadcaster"]
        OBS["OBS / Streamlabs<br/>(RTMP/SRT Output)"]
    end

    subgraph Ingest["Ingest Layer (Geo-Distributed)"]
        ING1["Ingest PoP US-East<br/>(Auth + Demux)"]
        ING2["Ingest PoP EU-West"]
        ING3["Ingest PoP AP-South"]
    end

    subgraph Transcoding["Live Transcoding"]
        TC1["GPU Transcoder<br/>(1080p60 -> 4 renditions)"]
        TC2["GPU Transcoder<br/>(Standby / Other Stream)"]
        HEALTH["Health Monitor<br/>(Failover Detection)"]
    end

    subgraph Origin["Origin Layer"]
        ORIG["Origin Servers<br/>(Segment Storage + Playlist)"]
        VOD_REC["VOD Recorder<br/>(Capture to S3)"]
    end

    subgraph CDN["CDN (Multi-Tier)"]
        EDGE1["Edge PoP 1<br/>(Cache + Serve)"]
        EDGE2["Edge PoP 2"]
        EDGEN["Edge PoP N<br/>(~200 locations)"]
    end

    subgraph Viewers["Viewers"]
        V1["Web Viewer"]
        V2["Mobile Viewer"]
        V3["TV Viewer"]
    end

    subgraph Chat["Chat System"]
        WSGW["WebSocket Gateway<br/>(Connection Management)"]
        CHATSVC["Chat Service<br/>(Validation + Moderation)"]
        REDIS["Redis Streams<br/>(Per-Channel Pub/Sub)"]
        AUTOMOD["AutoMod<br/>(ML Toxicity Filter)"]
    end

    subgraph Backend["Backend Services"]
        CHANNEL["Channel Service<br/>(PostgreSQL)"]
        DISC["Discovery Service<br/>(Browse + Search)"]
        VIEWER_CT["Viewer Count<br/>(Redis Approximate)"]
        CLIP["Clip Service"]
    end

    subgraph Storage["Storage"]
        S3["S3<br/>(VOD Archive)"]
        CASS["Cassandra<br/>(Chat Archive)"]
    end

    OBS -->|"RTMP/SRT"| ING1
    OBS -.->|"backup feed"| ING2

    ING1 -->|"raw stream"| TC1
    ING2 -.->|"backup"| TC2
    HEALTH -->|"monitor"| TC1
    HEALTH -->|"failover"| TC2

    TC1 -->|"HLS segments (4 renditions)"| ORIG
    ORIG -->|"push popular"| EDGE1
    ORIG -->|"push popular"| EDGE2
    ORIG -->|"on-demand"| EDGEN
    ORIG -->|"record"| VOD_REC
    VOD_REC --> S3

    EDGE1 -->|"HLS stream"| V1
    EDGE2 -->|"HLS stream"| V2
    EDGEN -->|"HLS stream"| V3

    V1 <-->|"WebSocket"| WSGW
    V2 <-->|"WebSocket"| WSGW
    WSGW --> CHATSVC
    CHATSVC --> AUTOMOD
    CHATSVC --> REDIS
    REDIS -->|"fan-out"| WSGW
    REDIS -->|"archive"| CASS

    V1 -->|"view events"| VIEWER_CT
    DISC --> CHANNEL
```

## 2. Deep-Dive: Live Video Pipeline (Ingest to Viewer)

```mermaid
flowchart TB
    subgraph BroadcasterSide["Broadcaster Side"]
        CAM["Camera/Screen Capture"]
        ENCODE_LOCAL["Local Encoder (OBS)<br/>H.264, 1080p60, 6 Mbps"]
        RTMP_OUT["RTMP Push to Ingest URL"]
    end

    subgraph IngestServer["Ingest Server"]
        AUTH["Stream Key Auth<br/>(Validate against DB)"]
        DEMUX["Demux RTMP<br/>(Separate Audio + Video)"]
        PROBE["Quality Probe<br/>(Bitrate, Resolution, FPS)"]
        FWD["Forward to Transcoder<br/>(Internal RTP/SRT)"]
    end

    subgraph LiveTranscoder["GPU Transcoder"]
        DECODE["Decode H.264<br/>(GPU NVDEC)"]
        subgraph Renditions["Parallel Encode (NVENC)"]
            ENC1080["1080p60 @ 6 Mbps<br/>(Passthrough or re-encode)"]
            ENC720["720p30 @ 2.5 Mbps"]
            ENC480["480p30 @ 1 Mbps"]
            ENC360["360p30 @ 600 Kbps"]
        end
        SEGMENT["HLS Segmenter<br/>(2s segments, aligned keyframes)"]
        subgraph LowLatency["Low-Latency Mode"]
            PARTIAL["LL-HLS Partial Segments<br/>(0.5s chunks)"]
        end
    end

    subgraph Distribution["Distribution"]
        PLAYLIST["Update .m3u8 Playlist<br/>(Add new segment reference)"]
        ORIGIN_STORE["Origin: Store Segments"]
        subgraph CDNFanOut["CDN Fan-Out"]
            PUSH_HOT["Proactive Push<br/>(Top Streams > 100K viewers)"]
            PULL_LONG["Pull on Demand<br/>(Smaller Streams)"]
        end
    end

    subgraph ViewerSide["Viewer Side"]
        PLAYER["Video Player"]
        ABR_ALGO["ABR Algorithm<br/>(Buffer-based / Throughput)"]
        RENDER["Decode + Render"]
    end

    CAM --> ENCODE_LOCAL --> RTMP_OUT
    RTMP_OUT -->|"RTMP over TLS"| AUTH
    AUTH -->|"valid"| DEMUX
    DEMUX --> PROBE
    PROBE --> FWD

    FWD --> DECODE
    DECODE --> ENC1080
    DECODE --> ENC720
    DECODE --> ENC480
    DECODE --> ENC360

    ENC1080 --> SEGMENT
    ENC720 --> SEGMENT
    ENC480 --> SEGMENT
    ENC360 --> SEGMENT
    SEGMENT --> PARTIAL

    SEGMENT --> PLAYLIST
    SEGMENT --> ORIGIN_STORE
    PARTIAL --> PLAYLIST

    ORIGIN_STORE --> PUSH_HOT
    ORIGIN_STORE --> PULL_LONG

    PUSH_HOT --> PLAYER
    PULL_LONG --> PLAYER
    PLAYER --> ABR_ALGO
    ABR_ALGO --> RENDER
```

## 3. Critical Path Sequence: Viewer Joins Live Stream

```mermaid
sequenceDiagram
    participant V as Viewer (Browser)
    participant API as API Gateway
    participant CH as Channel Service
    participant CDN as CDN Edge PoP
    participant ORIG as Origin Server
    participant WS as WebSocket Gateway
    participant REDIS as Redis Streams
    participant CHAT as Chat Service

    V->>API: GET /channels/ninja/playback
    API->>CH: Lookup channel status
    CH-->>API: {status: "live", viewer_count: 245000}
    API-->>V: {manifest_url, ws_chat_url, viewer_count}

    par Video + Chat connection
        V->>CDN: GET /live/ninja/master.m3u8
        CDN->>CDN: Cache hit (playlist updated every 2s)
        CDN-->>V: Master playlist (4 renditions)

        V->>WS: Connect WebSocket (channel=ninja)
        WS->>WS: Add connection to channel room
        WS->>REDIS: Subscribe to stream:chat:ninja
    end

    V->>V: Select initial rendition (720p based on bandwidth estimate)
    V->>CDN: GET /live/ninja/720p/segment_4521.ts
    CDN->>CDN: Cache hit (segment cached from origin push)
    CDN-->>V: Video segment (2s, 720p)

    V->>V: Decode + render first frame
    Note over V: Playback starts (< 2s from click)

    loop Every 2 seconds
        V->>CDN: GET /live/ninja/720p/segment_N.ts
        CDN-->>V: Next segment
        V->>V: ABR check: bandwidth OK -> stay at 720p
    end

    Note over V,REDIS: Chat message flow

    V->>WS: Send "Let's gooo!"
    WS->>CHAT: Validate message
    CHAT->>CHAT: AutoMod check (pass)
    CHAT->>CHAT: Rate limit check (pass)
    CHAT->>REDIS: XADD stream:chat:ninja {user, message}

    REDIS-->>WS: New messages (batched every 100ms)
    WS-->>V: WebSocket frame: [{user: "fan1", msg: "GG"}, {user: "viewer", msg: "Let's gooo!"}]

    Note over V: Viewer sees their message + others in chat

    V->>API: POST /channels/ninja/clip (last 30s)
    API->>API: Extract segments from VOD buffer, trim, store
    API-->>V: {clip_url: "https://clips.example.com/abc123"}
```
