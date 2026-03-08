# Zoom / Video Conferencing -- Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Devices"]
        WEB[Web Browser<br/>WebRTC]
        MOB[Mobile App<br/>iOS/Android]
        DESK[Desktop App<br/>Windows/Mac]
    end

    subgraph Edge["Edge / Connectivity"]
        LB_SIG[L7 Load Balancer<br/>Signaling]
        LB_MEDIA[L4 Load Balancer<br/>Media]
        STUN[STUN Servers]
        TURN[TURN Relay Servers]
    end

    subgraph Signaling["Signaling Plane"]
        SIG1[Signaling Server 1]
        SIG2[Signaling Server 2]
        SIG_N[Signaling Server N]
    end

    subgraph MediaPlane["Media Plane"]
        MR[Media Router<br/>SFU Assignment]
        subgraph SFU_Cluster_US["SFU Cluster US-East"]
            SFU1[SFU Server 1]
            SFU2[SFU Server 2]
        end
        subgraph SFU_Cluster_EU["SFU Cluster EU-West"]
            SFU3[SFU Server 3]
            SFU4[SFU Server 4]
        end
        SFU1 <-->|Cascade link<br/>inter-region| SFU3
    end

    subgraph Services["Application Services"]
        MEET[Meeting Service<br/>CRUD, auth]
        CHAT[Chat Service<br/>in-meeting messaging]
        REC[Recording Service<br/>headless participant]
        TRANS[Transcription Service<br/>speech-to-text]
    end

    subgraph Storage["Data Stores"]
        PG[PostgreSQL<br/>Meetings, Users]
        REDIS[Redis<br/>Meeting state, SFU load]
        S3[S3 / Object Storage<br/>Recordings]
        CDN_OUT[CDN<br/>Recording playback]
    end

    WEB & MOB & DESK -->|WSS signaling| LB_SIG
    LB_SIG --> SIG1 & SIG2 & SIG_N
    SIG1 & SIG2 & SIG_N -->|Meeting state| REDIS
    SIG1 & SIG2 & SIG_N -->|SDP/ICE relay| MR

    WEB & MOB & DESK -->|SRTP/UDP media| LB_MEDIA
    WEB & MOB & DESK -->|NAT discovery| STUN
    WEB & MOB & DESK -->|Relay fallback| TURN
    LB_MEDIA --> SFU1 & SFU2 & SFU3 & SFU4

    MR -->|Assign SFU| REDIS
    MR -->|Query load| SFU_Cluster_US & SFU_Cluster_EU

    MEET -->|CRUD| PG
    REC -->|Join as participant| SFU1
    REC -->|Upload recording| S3
    TRANS -->|Process audio| REC
    S3 --> CDN_OUT

    CHAT -->|Messages| REDIS
```

## 2. Deep-Dive: SFU Simulcast and Cascading Subsystem

```mermaid
flowchart TB
    subgraph Sender["Sender (Participant A)"]
        CAM[Camera Capture]
        ENC_H[Encoder: 720p 30fps<br/>1.5 Mbps]
        ENC_M[Encoder: 360p 15fps<br/>500 Kbps]
        ENC_L[Encoder: 180p 7fps<br/>150 Kbps]
    end

    subgraph SFU_Primary["SFU Server (US-East)"]
        RX[RTP Receiver]
        JB[Jitter Buffer<br/>50ms depth]
        NACK_BUF[NACK Buffer<br/>1s history]
        LAYER_SEL[Layer Selector<br/>per-receiver decision]
        BWE[Bandwidth Estimator<br/>TWcc / GCC algorithm]
        SPK[Active Speaker<br/>Detector]
        FWD[Packet Forwarder]
    end

    subgraph Cascade["Cascade to EU SFU"]
        CASCADE_LINK[Inter-SFU Link<br/>Private backbone]
        SFU_EU[SFU Server EU-West]
    end

    subgraph Receivers["Receivers"]
        RX_B[Participant B<br/>Good bandwidth<br/>Receives: 720p]
        RX_C[Participant C<br/>Mobile / poor network<br/>Receives: 180p]
        RX_D[Participant D<br/>Gallery view<br/>Receives: 360p]
        RX_E[Participant E<br/>EU region<br/>via cascade SFU]
    end

    subgraph Feedback["RTCP Feedback Loop"]
        REMB[Receiver BW Estimate]
        TWCC[Transport-Wide CC<br/>packet arrival reports]
        RTCP_RR[Receiver Report<br/>loss, jitter stats]
    end

    CAM --> ENC_H & ENC_M & ENC_L
    ENC_H -->|High layer RTP| RX
    ENC_M -->|Medium layer RTP| RX
    ENC_L -->|Low layer RTP| RX

    RX --> JB
    JB --> NACK_BUF
    JB --> LAYER_SEL
    SPK -->|Speaker info| LAYER_SEL
    BWE -->|Available BW per receiver| LAYER_SEL

    LAYER_SEL -->|Select high| FWD
    LAYER_SEL -->|Select medium| FWD
    LAYER_SEL -->|Select low| FWD

    FWD -->|720p| RX_B
    FWD -->|180p| RX_C
    FWD -->|360p| RX_D
    FWD -->|All layers| CASCADE_LINK
    CASCADE_LINK --> SFU_EU
    SFU_EU -->|Selected layer| RX_E

    RX_B & RX_C & RX_D -->|RTCP| TWCC & REMB & RTCP_RR
    TWCC & REMB & RTCP_RR --> BWE

    NACK_BUF -->|Retransmit on NACK| FWD
```

## 3. Critical Path Sequence: Joining a Meeting and Establishing Media

```mermaid
sequenceDiagram
    participant Client as Client App
    participant LB as Load Balancer
    participant MeetSvc as Meeting Service
    participant Sig as Signaling Server
    participant Redis as Redis
    participant MR as Media Router
    participant STUN_S as STUN Server
    participant SFU as SFU Server

    Client->>LB: POST /meetings/{id}/join<br/>{user_id, display_name}
    LB->>MeetSvc: Forward join request
    MeetSvc->>MeetSvc: Validate meeting password,<br/>check waiting room
    MeetSvc->>Redis: Get meeting:{id}:sfu assignment

    alt No SFU assigned yet (first joiner)
        MeetSvc->>MR: Assign SFU for meeting
        MR->>Redis: Query sfu:*:load for region
        MR->>Redis: Set meeting:{id}:sfu = sfu-us-east-7
        MR-->>MeetSvc: sfu-us-east-7
    end

    MeetSvc-->>Client: {session_token, signaling_ws_url,<br/>ice_servers, turn_servers}

    Client->>Sig: WSS CONNECT + JOIN_ROOM<br/>{meeting_id, session_token}
    Sig->>Redis: Add participant to meeting:{id}:participants
    Sig->>Sig: Notify existing participants<br/>of new joiner

    par ICE Gathering
        Client->>STUN_S: STUN Binding Request
        STUN_S-->>Client: Public IP:port (server-reflexive candidate)
        Client->>Client: Gather host candidates
    end

    Client->>Sig: SDP OFFER<br/>{audio: opus, video: VP8 simulcast 3 layers}
    Sig->>SFU: Forward SDP OFFER
    SFU->>SFU: Allocate media session,<br/>prepare DTLS fingerprint
    SFU-->>Sig: SDP ANSWER<br/>{accepted codecs, ICE candidates}
    Sig-->>Client: SDP ANSWER

    Client->>SFU: ICE Connectivity Check (STUN binding)
    SFU-->>Client: ICE Success

    Client->>SFU: DTLS Handshake<br/>(establish SRTP keys)
    SFU-->>Client: DTLS Complete

    Note over Client,SFU: Media session established

    Client->>SFU: SRTP Audio (Opus, 48kbps)
    Client->>SFU: SRTP Video High (720p, 1.5Mbps)
    Client->>SFU: SRTP Video Medium (360p, 500kbps)
    Client->>SFU: SRTP Video Low (180p, 150kbps)

    SFU->>SFU: Select appropriate layer<br/>per receiver based on BWE
    SFU->>Client: SRTP forwarded streams<br/>from other participants

    loop Every 5 seconds
        Client->>SFU: RTCP Receiver Report<br/>(loss, jitter, RTT)
        SFU->>SFU: Update BWE per receiver
        SFU->>SFU: Adjust layer selection
    end

    Note over Client,SFU: Total join-to-first-frame: ~2-3 seconds
```
