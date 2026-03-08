# Proximity Service (Nearby Friends) -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Mobile Clients"]
        UserA["User A<br/>(iOS/Android)"]
        UserB["User B<br/>(iOS/Android)"]
        UserN["User N<br/>(iOS/Android)"]
    end

    subgraph WebSocketLayer["WebSocket Gateway Layer"]
        WS1["WS Gateway 1<br/>(100K connections)"]
        WS2["WS Gateway 2<br/>(100K connections)"]
        WSN["WS Gateway N<br/>(100K connections)"]
        ConnRegistry["Connection Registry<br/>(Redis: user_id -> gateway)"]
    end

    subgraph LocationLayer["Location Layer"]
        LocIngestion["Location Ingestion<br/>Service"]
        LocCache["Location Cache<br/>(Redis, 320 MB)"]
    end

    subgraph PubSubLayer["Pub/Sub Layer"]
        PS1["Pub/Sub Server 1"]
        PS2["Pub/Sub Server 2"]
        PSN["Pub/Sub Server N"]
        note1["Consistent Hash Ring:<br/>user_id -> Pub/Sub server"]
    end

    subgraph ProximityLayer["Proximity Computation"]
        ProxWorker["Proximity Workers<br/>(Geohash Filter +<br/>Distance Calculation)"]
    end

    subgraph SupportServices["Support Services"]
        FriendSvc["Friendship Service<br/>(Friend Lists)"]
        PrivacySvc["Privacy Service<br/>(Visibility Rules)"]
        NotifSvc["Notification Service<br/>(Push Notifications)"]
    end

    subgraph DataStores["Data Stores"]
        FriendDB["Friendship Graph<br/>(Sharded KV Store)"]
        SettingsDB["Privacy Settings<br/>(PostgreSQL)"]
        FriendCache["Friend List Cache<br/>(Redis)"]
    end

    UserA -->|WebSocket| WS1
    UserB -->|WebSocket| WS2
    UserN -->|WebSocket| WSN

    WS1 --> ConnRegistry
    WS2 --> ConnRegistry
    WSN --> ConnRegistry

    WS1 -->|location updates| LocIngestion
    WS2 -->|location updates| LocIngestion
    WSN -->|location updates| LocIngestion

    LocIngestion -->|write position| LocCache
    LocIngestion -->|publish update| PS1
    LocIngestion -->|publish update| PS2
    LocIngestion -->|publish update| PSN

    PS1 -->|deliver to subscribers| ProxWorker
    PS2 -->|deliver to subscribers| ProxWorker
    PSN -->|deliver to subscribers| ProxWorker

    ProxWorker -->|read friend location| LocCache
    ProxWorker -->|check friend list| FriendSvc
    ProxWorker -->|check privacy| PrivacySvc
    ProxWorker -->|nearby alert| ConnRegistry
    ConnRegistry -->|route to gateway| WS1
    ConnRegistry -->|route to gateway| WS2

    FriendSvc --> FriendDB
    FriendSvc --> FriendCache
    PrivacySvc --> SettingsDB

    ProxWorker -->|push notification| NotifSvc
```

## 2. Deep-Dive: Pub/Sub Channel and Proximity Computation

```mermaid
flowchart TB
    subgraph UserOnline["User A Comes Online"]
        AOnline["User A connects<br/>(WebSocket)"]
        FetchFriends["Fetch A's friend list<br/>(500 friends)"]
        FilterOnline["Filter to online friends<br/>(~50 friends)"]
        Subscribe["Subscribe A to<br/>each online friend's channel"]
        NotifyFriends["Notify each online friend<br/>to subscribe to A's channel"]
    end

    subgraph LocationUpdate["User A Sends Location Update"]
        AUpdate["A sends: lat=40.7, lng=-74.0"]
        GeohashCalc["Compute A's geohash:<br/>'dr5ru7'"]
        UpdateCache["Write to Location Cache:<br/>A -> (40.7, -74.0, dr5ru7)"]
        PublishChannel["Publish to A's channel<br/>on Pub/Sub server H(A)"]
    end

    subgraph SubscriberProcessing["Each Subscriber (Friend B) Processes"]
        ReceiveUpdate["B's proximity worker<br/>receives A's location"]
        CoarseFilter["Geohash coarse filter:<br/>A='dr5ru7', B='dr5ru6'<br/>Adjacent? YES"]
        ExactDistance["Compute Haversine distance:<br/>d(A, B) = 0.8 km"]
        CheckThreshold["B's proximity radius = 5 km<br/>0.8 km < 5 km? YES"]
        CheckPrivacy["Privacy check:<br/>A allows B to see? YES"]
        SendNotif["Push to B via WebSocket:<br/>'Friend A is 0.8 km away'"]
    end

    subgraph SkipPath["Skip Path (Distant Friend C)"]
        ReceiveUpdateC["C's proximity worker<br/>receives A's location"]
        CoarseFilterC["Geohash coarse filter:<br/>A='dr5ru7', C='9q8yy'<br/>Adjacent? NO"]
        SkipC["SKIP distance calculation<br/>(90%+ of cases)"]
    end

    AOnline --> FetchFriends --> FilterOnline --> Subscribe --> NotifyFriends

    AUpdate --> GeohashCalc --> UpdateCache --> PublishChannel

    PublishChannel --> ReceiveUpdate
    ReceiveUpdate --> CoarseFilter
    CoarseFilter -->|adjacent cells| ExactDistance
    ExactDistance --> CheckThreshold
    CheckThreshold -->|within radius| CheckPrivacy
    CheckPrivacy -->|allowed| SendNotif

    PublishChannel --> ReceiveUpdateC
    ReceiveUpdateC --> CoarseFilterC
    CoarseFilterC -->|distant cells| SkipC
```

## 3. Critical Path Sequence: Friend Proximity Detection

```mermaid
sequenceDiagram
    participant A as User A (Mobile)
    participant WSA as WS Gateway (A)
    participant LI as Location Ingestion
    participant LC as Location Cache
    participant PS as Pub/Sub Server H(A)
    participant PW as Proximity Worker (B)
    participant FS as Friendship Service
    participant PR as Privacy Service
    participant WSB as WS Gateway (B)
    participant B as User B (Mobile)

    Note over A,B: Setup: A and B are friends. Both online.
    Note over A,B: B is subscribed to A's Pub/Sub channel.

    A->>WSA: Location update: (40.748, -73.986)
    WSA->>LI: Forward location update
    LI->>LC: GEOADD user:A 40.748 -73.986
    LC-->>LI: OK

    LI->>PS: Publish to channel "user:A"<br/>{user_id: A, lat: 40.748, lng: -73.986, geohash: "dr5ru7"}

    PS->>PW: Deliver to subscriber (B's worker)

    PW->>LC: GET location of user:B
    LC-->>PW: B at (40.752, -73.978), geohash: "dr5ru6"

    PW->>PW: Geohash coarse filter:<br/>"dr5ru7" adjacent to "dr5ru6"? YES

    PW->>PW: Haversine distance:<br/>d = 0.85 km

    PW->>PR: Can A be seen by B?
    PR-->>PW: YES (mutual sharing enabled)

    PW->>PW: B's radius = 5 km<br/>0.85 km < 5 km -> NEARBY

    PW->>WSB: Push notification for user B<br/>{type: "friend_nearby", friend: A, distance: 0.85}

    WSB->>B: WebSocket push:<br/>"Friend A is 0.85 km away"

    Note over A,B: End-to-end latency: ~2-5 seconds
```
