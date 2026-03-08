# 24-Hour Story Feature — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    Client[Mobile Client]

    subgraph Gateway["API Layer"]
        LB[Load Balancer]
        AG[API Gateway]
    end

    subgraph StoryServices["Story Services"]
        SS[Story Service<br/>CRUD + TTL Logic]
        TS[Tray Service<br/>Personalized Ordering]
        VS[View Tracking Service]
        MPS[Media Processing<br/>Workers]
    end

    subgraph CacheLayer["Cache Layer"]
        REDIS[(Redis Cluster<br/>Tray Cache<br/>View Counts<br/>Seen/Unseen State)]
    end

    subgraph DataStores["Data Stores"]
        CASS[(Cassandra Cluster<br/>Story Items TTL=48h<br/>Views TTL=48h)]
        S3[(S3 Storage<br/>Lifecycle: delete 48h)]
    end

    subgraph EventBus["Event Bus"]
        K[Kafka]
        TW[Tray Update Workers]
        EW[Expiration Watchdog]
    end

    CDN[CDN<br/>max-age: 24h]

    Client -->|API requests| LB
    Client -->|Direct upload| S3
    Client -->|View media| CDN
    LB --> AG
    AG --> SS & TS & VS

    SS -->|Write with TTL| CASS
    SS -->|Pre-signed URL| S3
    SS -->|Publish StoryCreated| K

    TS -->|Read tray| REDIS
    TS -->|Fallback| CASS

    VS -->|Dedup check| REDIS
    VS -->|Write view| CASS
    VS -->|Increment count| REDIS

    K --> TW
    TW -->|Update follower trays| REDIS
    K --> MPS
    MPS -->|Process media| S3

    EW -->|Clean expired entries| REDIS

    S3 -->|Origin| CDN
```

## 2. Story Tray Generation — Deep Dive

```mermaid
flowchart TB
    subgraph StoryCreation["Story Created"]
        NEW[User A posts story]
        EV[StoryCreated Event<br/>to Kafka]
    end

    subgraph TrayFanOut["Fan-out-on-Write"]
        KC[Kafka Consumer]
        FL[Get A's Followers]
        CT{Celebrity<br/>check}
        NORM[Normal Path:<br/>Update each<br/>follower's tray]
        CELEB[Celebrity Path:<br/>Skip fan-out,<br/>pull at read time]
    end

    subgraph TrayCache["Redis Tray Cache"]
        T1["ZADD tray:follower_1<br/>score=timestamp<br/>member=user_A_data"]
        T2["ZADD tray:follower_2"]
        TN["ZADD tray:follower_N"]
    end

    subgraph TrayRead["Tray Read Path"]
        REQ[User opens app]
        RC[Read tray:{user_id}<br/>from Redis]
        EXPIRE[Filter expired<br/>entries: expires_at < now]
        CPULL[Pull celebrity<br/>stories]
        MERGE[Merge + Deduplicate]
        UNSEEN[Split: unseen first<br/>then seen]
        AFFINITY[Sort by affinity<br/>within each group]
        RESP[Return ordered tray]
    end

    subgraph SeenTracking["Seen/Unseen"]
        VIEW[User views story]
        MARK[Set bit in<br/>seen:{user_id}:{date}]
        CHECK[On tray build:<br/>check all items seen?]
    end

    NEW --> EV
    EV --> KC
    KC --> FL
    FL --> CT
    CT -->|"< 100K followers"| NORM
    CT -->|">= 100K followers"| CELEB
    NORM --> T1 & T2 & TN

    REQ --> RC
    RC --> EXPIRE
    EXPIRE --> CPULL
    CPULL --> MERGE
    MERGE --> UNSEEN
    UNSEEN --> AFFINITY
    AFFINITY --> RESP

    VIEW --> MARK
    MARK --> CHECK
    CHECK --> UNSEEN
```

## 3. Story Upload, View, and Expiration — Sequence Diagram

```mermaid
sequenceDiagram
    participant A as Author
    participant AG as API Gateway
    participant SS as Story Service
    participant S3 as S3 Storage
    participant CASS as Cassandra
    participant K as Kafka
    participant TW as Tray Worker
    participant R as Redis
    participant CDN as CDN
    participant V as Viewer

    Note over A,V: --- Story Upload Phase ---

    A->>AG: POST /v1/stories/upload-url
    AG->>SS: Request pre-signed URL
    SS->>S3: Generate pre-signed URL (24h expiry)
    S3-->>SS: Signed URL
    SS-->>A: Signed URL

    A->>S3: PUT media (direct upload)
    S3-->>A: 200 OK

    A->>AG: POST /v1/stories (metadata)
    AG->>SS: Create story item
    SS->>CASS: INSERT story (TTL=48h)
    SS->>K: Publish StoryCreated
    SS-->>A: 201 {story_item_id, expires_at}

    K->>TW: Consume StoryCreated
    TW->>TW: Get author's followers
    loop Each follower batch
        TW->>R: ZADD tray:{follower_id} timestamp author_data
    end

    Note over A,V: --- Story View Phase ---

    V->>AG: GET /v1/stories/tray
    AG->>R: Read tray:{viewer_id}
    R-->>AG: Sorted tray entries
    AG->>AG: Filter expired, split seen/unseen
    AG-->>V: Ordered story tray

    V->>AG: GET /v1/stories/users/{author_id}
    AG->>SS: Get active stories
    SS->>CASS: SELECT WHERE user_id=author AND expires_at > now
    CASS-->>SS: story items[]
    SS-->>V: Story items with CDN URLs

    V->>CDN: GET media
    CDN-->>V: Media content

    V->>AG: POST /v1/stories/{id}/view
    AG->>R: PFADD views:{id} viewer_id (dedup)
    AG->>CASS: INSERT view (TTL=48h)
    AG->>R: INCR view_count:{id}

    Note over A,V: --- Expiration Phase (24h later) ---

    Note right of SS: Application: expires_at check returns 404
    Note right of CDN: CDN: max-age expired, removes from cache
    Note right of S3: S3: lifecycle policy deletes object at 48h
    Note right of CASS: Cassandra: row TTL expires, tombstoned
    Note right of R: Redis: expiration watchdog removes tray entry
```
