# System Design Interview: Zoom / Video Conferencing -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a video conferencing system like Zoom that supports real-time audio/video communication for meetings of varying sizes -- from 1-on-1 calls to webinars with thousands of viewers. The core challenges are achieving sub-200ms end-to-end latency for interactive video, handling heterogeneous network conditions across participants, and scaling media routing infrastructure globally. I will cover the signaling layer, media transport architecture (SFU vs MCU), adaptive bitrate, and recording. Let me begin with some clarifying questions."

## 1) Clarifying Questions (2-5 min)

**Scope**
- What meeting sizes do we support? *1-on-1 up to 1,000-participant meetings, plus webinar mode up to 10,000 viewers.*
- Do we need screen sharing, chat, reactions, and virtual backgrounds? *Yes -- screen sharing and in-meeting chat are core; virtual backgrounds are a stretch goal.*
- Do we need recording and playback? *Yes -- cloud recording with post-meeting playback.*

**Scale**
- How many concurrent meetings? *Target 1 million concurrent meetings with an average of 5 participants each = 5 million concurrent media streams.*
- What is peak concurrent users? *10 million users in meetings simultaneously.*
- Geographic distribution? *Global -- users in North America, Europe, Asia, and more.*

**Policy / Constraints**
- End-to-end latency target? *p95 < 150ms for audio, p95 < 200ms for video.*
- Do we need end-to-end encryption (E2EE)? *Optional mode for enterprise customers.*
- Supported platforms? *Web (WebRTC), iOS, Android, desktop apps.*

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Concurrent meetings | Given | 1M |
| Avg participants/meeting | Given | 5 |
| Concurrent participants | 1M x 5 | 5M |
| Video bitrate per stream | 720p adaptive | 1.5 Mbps avg |
| Audio bitrate per stream | Opus codec | 48 Kbps |
| Upstream per participant | 1 video + 1 audio | ~1.55 Mbps |
| Total upstream bandwidth | 5M x 1.55 Mbps | ~7.75 Tbps |
| Downstream per participant | (N-1) streams, avg 4 | ~6.2 Mbps |
| Total downstream bandwidth | 5M x 6.2 Mbps | ~31 Tbps |
| SFU servers needed | 500 streams per SFU | 10,000 SFU servers |
| Recording storage/hour | 1M meetings x 500 Kbps composite | ~225 TB/hour |
| Signaling QPS | 5M users x 1 msg/10s | 500K QPS |
| Meeting metadata storage | 1M meetings x 5KB | 5 GB (trivial) |

**Key insight:** The dominant cost is media bandwidth, not compute or storage. Global PoP distribution is essential to minimize transit costs and latency.

## 3) Requirements (3-5 min)

### Functional
- **Join/leave meetings**: Users join via link or meeting ID with optional password.
- **Real-time audio/video**: Bidirectional media streams with < 200ms latency.
- **Screen sharing**: Share entire screen or specific application window.
- **In-meeting chat**: Text chat with file sharing within the meeting.
- **Recording**: Cloud-based recording with automatic transcription.
- **Meeting controls**: Mute/unmute, camera on/off, host controls (admit from waiting room, kick, mute all).

### Non-functional
- **Ultra-low latency**: p95 audio < 150ms, p95 video < 200ms glass-to-glass.
- **High availability**: 99.99% uptime for signaling; media path tolerates individual server failures.
- **Scalability**: 1M concurrent meetings, 5M concurrent participants.
- **Adaptive quality**: Graceful degradation on poor networks (reduce resolution, frame rate, switch to audio-only).
- **Global reach**: Media servers in 20+ regions to minimize geographic latency.
- **Security**: Encrypted media transport; optional E2EE for enterprise.

## 4) API Design (2-4 min)

### Signaling API (WebSocket + REST)

```
POST /api/v1/meetings
  Body: {host_id, title, scheduled_time, duration, password, settings}
  Returns: {meeting_id, join_url, host_key}

POST /api/v1/meetings/{meeting_id}/join
  Body: {user_id, display_name, audio_only}
  Returns: {session_token, signaling_ws_url, turn_servers, ice_servers}

# WebSocket signaling messages
JOIN_ROOM {meeting_id, session_token}
OFFER {sdp, target_user_id}
ANSWER {sdp, target_user_id}
ICE_CANDIDATE {candidate, target_user_id}
MEDIA_STATE {audio: bool, video: bool, screen_share: bool}
LEAVE_ROOM {}

# Host controls
POST /api/v1/meetings/{meeting_id}/mute-all
POST /api/v1/meetings/{meeting_id}/participants/{user_id}/remove
POST /api/v1/meetings/{meeting_id}/recording/start
POST /api/v1/meetings/{meeting_id}/recording/stop
```

### Media Transport
```
# SRTP/DTLS over UDP for audio/video
# SCTP over DTLS for data channel (chat, reactions)
# TURN relay as fallback when P2P/direct fails
```

## 5) Data Model (3-5 min)

### Meetings Table (PostgreSQL)
| Column | Type | Notes |
|--------|------|-------|
| meeting_id | UUID | PK |
| host_user_id | UUID | FK to users |
| title | VARCHAR(255) | |
| password_hash | VARCHAR | Optional meeting password |
| scheduled_start | TIMESTAMP | |
| actual_start | TIMESTAMP | |
| actual_end | TIMESTAMP | |
| status | ENUM | scheduled, active, ended |
| settings | JSONB | waiting_room, mute_on_entry, etc. |
| sfu_server_id | VARCHAR | Assigned SFU cluster |
| recording_enabled | BOOLEAN | |
| created_at | TIMESTAMP | |

### Participants Table (PostgreSQL + Redis)
| Column | Type | Notes |
|--------|------|-------|
| meeting_id | UUID | PK (composite) |
| user_id | UUID | PK (composite) |
| display_name | VARCHAR | |
| join_time | TIMESTAMP | |
| leave_time | TIMESTAMP | |
| role | ENUM | host, co-host, participant |
| audio_state | BOOLEAN | In Redis for real-time |
| video_state | BOOLEAN | In Redis for real-time |

### Recordings Table (PostgreSQL + S3)
| Column | Type | Notes |
|--------|------|-------|
| recording_id | UUID | PK |
| meeting_id | UUID | FK |
| s3_path | VARCHAR | Path to recorded file |
| duration_sec | INT | |
| file_size_bytes | BIGINT | |
| format | VARCHAR | mp4 |
| transcript_path | VARCHAR | Path to transcript file |
| status | ENUM | recording, processing, ready, failed |
| created_at | TIMESTAMP | |

### Real-time State (Redis)
| Key | Value | Notes |
|-----|-------|-------|
| `meeting:{id}:participants` | Hash of user_id -> media state | Active participant state |
| `meeting:{id}:sfu` | SFU server assignment | Routing info |
| `sfu:{server_id}:load` | Current stream count | Load balancing |

## 6) High-Level Architecture (5-8 min)

**Dataflow:** A user clicks a join link, the client calls the Join API, which authenticates and returns a signaling WebSocket URL and ICE/TURN server details. The client connects to the signaling server and exchanges SDP offers/answers to negotiate media. Once negotiated, the client sends audio/video streams (SRTP over UDP) to the assigned SFU server. The SFU selectively forwards streams to other participants based on their subscription (active speaker, gallery view). The SFU adapts stream quality using simulcast layers based on each receiver's bandwidth.

**Components:**
- **Signaling Server**: Handles WebSocket connections for SDP exchange, ICE candidate relay, and meeting control messages. Stateless, horizontally scalable.
- **SFU (Selective Forwarding Unit)**: Receives media from each participant and selectively forwards to others. No transcoding -- just routing. Core of the media plane.
- **TURN Server**: Relay for participants behind restrictive NATs/firewalls where direct UDP fails.
- **Media Router**: Assigns meetings to SFU clusters based on participant geography and server load.
- **Recording Service**: Connects to the SFU as a headless participant, receives all streams, composites into a single video, and uploads to S3.
- **Meeting Service**: CRUD for meeting management, authentication, and authorization.
- **Chat Service**: In-meeting text chat via data channels or WebSocket fallback.
- **CDN**: Serves recorded meeting playback.

**One-sentence summary:** Clients negotiate sessions via a signaling server, then stream audio/video over SRTP/UDP to geographically-near SFU servers that selectively forward media to other participants, adapting quality via simulcast based on each receiver's network conditions.

## 7) Deep Dive #1: SFU Architecture and Simulcast (8-12 min)

### Why SFU over MCU or Mesh?

| Topology | Pros | Cons |
|----------|------|------|
| **Mesh (P2P)** | No server cost, lowest latency for 2 people | O(N^2) connections; unusable beyond 4-5 participants |
| **MCU (Mixing)** | Single stream downstream per participant | Enormous server-side CPU for transcoding; adds latency |
| **SFU (Forwarding)** | No transcoding, low server CPU, per-receiver quality adaptation | Downstream bandwidth = O(N) per participant |

We use **SFU** as the primary architecture because it balances server cost (no transcoding) with scalability (supports dozens of participants per meeting). For webinars (1,000+ viewers), we cascade SFUs hierarchically.

### Simulcast

Each sender publishes 3 simultaneous video layers:
- **High**: 720p at 30fps (~1.5 Mbps)
- **Medium**: 360p at 15fps (~500 Kbps)
- **Low**: 180p at 7fps (~150 Kbps)

The SFU selects which layer to forward to each receiver based on:
1. **Receiver bandwidth**: Estimated via RTCP receiver reports and REMB/TWcc feedback.
2. **UI layout**: Active speaker gets high layer; gallery thumbnails get low layer.
3. **Explicit subscription**: Client signals which participants it wants at which quality.

### Bandwidth Estimation

The SFU uses **Transport-Wide Congestion Control (TWcc)**:
1. Each RTP packet gets a transport-wide sequence number.
2. The receiver sends periodic feedback listing received packets and their arrival times.
3. The SFU's congestion controller (GCC algorithm) estimates available bandwidth.
4. If bandwidth drops, the SFU:
   - First: switches to lower simulcast layer
   - Then: reduces frame rate via temporal layer selection
   - Finally: drops video entirely and sends audio-only

### SFU Cascading for Large Meetings

For meetings with > 50 participants or geographically distributed attendees:
- Multiple SFU servers form a **cascade**. Each SFU handles a subset of participants (typically by region).
- SFUs forward streams to each other over dedicated inter-SFU links (high-bandwidth private network).
- A participant in Europe sends to the EU SFU, which cascades to the US SFU for American participants.
- The **Media Router** decides the cascade topology based on participant locations.

### Active Speaker Detection

The SFU analyzes audio levels from each participant:
- Audio energy is calculated from RTP audio frames (every 20ms).
- A smoothing algorithm (exponential moving average) prevents rapid speaker switching.
- The dominant speaker's video is forwarded at high quality; others at low quality.
- Speaker change events are sent to clients via the signaling channel.

### Handling Packet Loss

UDP has no built-in reliability. For media:
- **NACK**: Receiver requests retransmission of specific lost packets (good for low loss rates < 5%).
- **FEC (Forward Error Correction)**: Sender adds redundancy packets (good for bursty loss).
- **PLI (Picture Loss Indication)**: Receiver requests a new keyframe when too many packets are lost to reconstruct the frame.
- The SFU mediates NACKs -- it can serve retransmissions from its jitter buffer without bothering the original sender.

## 8) Deep Dive #2: Signaling and Session Establishment (5-8 min)

### WebRTC Signaling Flow

1. **Client A** calls the Join API and gets a signaling WebSocket URL.
2. Client A connects to the **Signaling Server** and sends `JOIN_ROOM`.
3. The Signaling Server notifies all existing participants that A has joined.
4. **Client A** creates an SDP offer (describing its media capabilities) and sends it to the Signaling Server.
5. The Signaling Server forwards the offer to the SFU (not to other participants -- this is a star topology).
6. The SFU generates an SDP answer and returns it via the Signaling Server.
7. Both sides exchange ICE candidates for NAT traversal.
8. DTLS handshake establishes encryption keys.
9. SRTP media begins flowing.

### ICE and NAT Traversal

Most users are behind NATs. The connectivity establishment process:
1. **STUN**: Client discovers its public IP/port via a STUN server.
2. **ICE**: Client gathers candidates (host, server-reflexive, relay) and tests connectivity.
3. **TURN**: If direct UDP fails (corporate firewalls), traffic is relayed through TURN servers.
4. Approximately 85% of connections succeed with STUN; 15% need TURN.

### Signaling Server Scalability

- Signaling servers are stateless -- meeting state lives in Redis.
- Any signaling server can handle any meeting.
- Load balanced via standard L7 LB with sticky sessions for WebSocket durability.
- At 500K QPS signaling, ~50 signaling servers (10K QPS each).

### Meeting Assignment

When a meeting starts, the Media Router selects an SFU cluster:
1. Identify the geographic region of the host (from IP geolocation).
2. Query SFU load balancer for available capacity in that region.
3. Assign the meeting to the least-loaded SFU in the nearest region.
4. Store the assignment in Redis: `meeting:{id}:sfu -> sfu-eu-west-3`.
5. When participants from other regions join, optionally cascade to their regional SFU.

## 9) Trade-offs and Alternatives (3-5 min)

### SFU vs MCU
- SFU is CPU-cheap but bandwidth-expensive (downstream = O(N) per user).
- MCU is CPU-expensive but bandwidth-cheap (downstream = 1 stream per user).
- For meetings < 50 people, SFU wins on latency and cost. For webinars with 10K viewers, a hybrid approach works: SFU for active speakers, MCU-composed stream for passive viewers.

### UDP vs TCP for Media
- UDP is preferred: no head-of-line blocking, allows custom congestion control.
- TCP fallback (WebSocket-tunneled media) for extreme firewall environments. Adds ~50ms latency and risks stalls from retransmission.

### WebRTC vs Custom Protocol
- WebRTC provides browser-native support, standardized codecs, and built-in encryption.
- Custom protocols (like Zoom's original) can optimize for specific use cases but require native apps.
- We use WebRTC for web clients and a custom transport layer for native apps that falls back to WebRTC semantics.

### CAP / PACELC
- **Meeting state**: CP -- participants must see consistent meeting state (who is in the meeting, who is muted). Served from Redis with single-leader writes.
- **Media plane**: Inherently AP -- a few dropped frames are acceptable; availability and low latency trump consistency.
- **Recording**: Eventually consistent -- recording can lag real-time by seconds without user impact.

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (10M concurrent meetings, 50M participants)
- **SFU fleet**: Scale to 100,000 SFU servers. Automate provisioning with Kubernetes and spot instances for non-recording workloads.
- **Regional expansion**: Add PoPs in underserved regions (Africa, South America, Southeast Asia).
- **Simulcast optimization**: Add a 4th layer (1080p) for high-bandwidth users; more aggressive downscaling for mobile.
- **Signaling**: Scale to 500 signaling servers. Consider gRPC for inter-service communication to reduce overhead.

### 100x (100M concurrent meetings, 500M participants)
- **Edge compute**: Deploy lightweight SFUs at ISP edge locations to minimize last-mile latency.
- **Codec innovation**: Adopt AV1/SVC for better compression ratios (30-50% bandwidth savings over VP8/H.264).
- **Hierarchical cascading**: 3-tier SFU cascade (edge -> regional -> backbone) to manage inter-region traffic.
- **Meeting sharding**: Split very large meetings across multiple SFU clusters with server-side mixing for overflow viewers.
- **Cost**: At 100x, bandwidth is the dominant cost (~$100M+/month). Private peering agreements and own backbone network (like Zoom Meeting Connector) become essential.

## 11) Reliability and Fault Tolerance (3-5 min)

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| SFU server crash | All participants in affected meetings lose media | Clients detect loss via ICE keepalives; auto-reconnect to standby SFU within 2-3s |
| Signaling server crash | New joins/leaves fail temporarily | Stateless servers behind LB; clients reconnect to another instance |
| TURN server crash | Relay-dependent users lose connectivity | Multiple TURN servers per region; client tries next candidate |
| Network partition between regions | Cross-region cascade breaks | Each region operates independently; cross-region participants rejoin local SFU |
| Recording service failure | Recording stops | Recording client reconnects; gap in recording is logged |

### SFU Failover Strategy
1. Each meeting has a primary and standby SFU pre-assigned.
2. The standby SFU receives a low-bitrate copy of all streams (warm standby).
3. On primary failure, the signaling server sends `RECONNECT` to all clients with the standby SFU's address.
4. Clients re-establish media within 2-3 seconds. Brief audio/video interruption.

### Graceful Degradation
- Under load: reduce max video quality from 720p to 360p globally.
- Under extreme load: disable video for new meetings, audio-only mode.
- Prioritize: enterprise/paid meetings over free-tier meetings.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Meeting join success rate** (target: > 99.5%) -- alert below 99%.
- **Media quality (MOS score)** -- Mean Opinion Score estimated from packet loss, jitter, latency. Alert if average MOS < 3.5.
- **Glass-to-glass latency** (p50, p95) -- alert if p95 > 300ms.
- **Packet loss rate** per SFU -- alert if > 2% sustained.
- **SFU CPU and bandwidth utilization** -- autoscale triggers at 70%.
- **TURN relay usage** -- track percentage of users needing TURN (expect ~15%).

### Dashboards
- Global meeting map: active meetings by region with health indicators.
- SFU fleet capacity: streams per server, bandwidth utilization heatmap.
- Quality of Experience (QoE): MOS score distribution, resolution distribution, audio-only fallback rate.

### Runbooks
- **High packet loss in region**: Check ISP peering; route through alternate backbone path.
- **SFU overloaded**: Trigger horizontal scaling; redistribute meetings.
- **TURN usage spike**: Investigate firewall changes at enterprise customers; scale TURN fleet.

## 13) Security (1-3 min)

- **Media encryption**: DTLS-SRTP for all media streams (mandatory). Encryption keys negotiated per-session via DTLS handshake.
- **E2EE (optional)**: Clients encrypt media with per-meeting keys before sending to SFU. SFU forwards encrypted payloads without decryption. Key exchange via out-of-band mechanism (e.g., MLS protocol). SFU can still route based on unencrypted RTP headers.
- **Signaling security**: WSS (TLS) for all signaling. JWT authentication with short-lived tokens.
- **Meeting security**: Waiting room, meeting passwords, host-only screen sharing controls.
- **Abuse prevention**: Rate limiting on meeting creation. Reporting mechanism for abusive participants. Watermarking for enterprise recordings.
- **Compliance**: SOC2, HIPAA (for healthcare customers). Data residency options (keep recordings in specific regions).

## 14) Team and Operational Considerations (1-2 min)

- **Team structure**: Media team (SFU, codecs, quality) -- 6-8 engineers. Signaling team (WebSocket, meeting management) -- 3-4 engineers. Infrastructure team (SFU fleet, TURN, networking) -- 4-5 engineers. Client SDK team (WebRTC integration per platform) -- 5-6 engineers.
- **Deployment**: SFU binary deploys via blue-green at the cluster level. New meetings go to new cluster; existing meetings drain from old cluster.
- **On-call**: 24/7 on-call with regional handoff. Tier 1 handles SFU scaling and routing. Tier 2 handles codec/quality issues and network debugging.
- **Rollout**: Launch with 1-on-1 and small meetings (< 10 people). Expand to large meetings after validating cascading. Webinar mode as a separate launch.

## 15) Common Follow-up Questions

**Q: How does screen sharing work technically?**
A: Screen capture is done client-side via browser `getDisplayMedia()` API or native screen capture APIs. The captured frames are encoded as a separate video track (typically at higher resolution but lower frame rate -- 1080p at 5-10fps). The SFU treats it as another video stream but with a "screen" content type hint, allowing receivers to optimize rendering (e.g., sharper text at lower frame rate).

**Q: How do you handle a meeting with 1,000 participants?**
A: With SFU cascading across 5-10 SFU servers (100-200 participants each). Only active speakers' video is forwarded at high quality. Gallery view shows the most recent N speakers. Participants not speaking send no video (or only low-quality thumbnails). Audio mixing is done: the SFU selects the top 3 audio streams by energy and forwards only those, mixing them server-side if needed.

**Q: How does cloud recording work without re-encoding?**
A: A recording bot joins as a headless participant on the SFU. It receives all media streams and writes raw RTP packets to disk. Post-meeting, a processing pipeline decodes and composites the streams into a single MP4 using ffmpeg. This avoids real-time transcoding costs. For live recording preview, a low-quality composite is generated in near-real-time.

**Q: How do you handle clock synchronization for lip sync?**
A: Audio and video have separate RTP streams with independent timestamps. The receiver uses RTCP Sender Reports to map RTP timestamps to NTP wall-clock time. The client's jitter buffer aligns audio and video frames using these synchronized timestamps, typically targeting 40ms audio-video sync tolerance.

## 16) Closing Summary (30-60s)

> "We designed a video conferencing system supporting 1M concurrent meetings with 5M participants, achieving sub-200ms glass-to-glass latency. The architecture uses SFU-based media routing with 3-layer simulcast for adaptive quality, WebRTC signaling for session establishment, and SFU cascading for large/geo-distributed meetings. Key design decisions include SFU over MCU to avoid transcoding costs, simulcast with TWcc-based bandwidth estimation for graceful quality adaptation, warm standby SFUs for sub-3-second failover, and hierarchical cascading for meetings exceeding 50 participants. The system degrades gracefully from 720p to 360p to audio-only as network conditions worsen, and scales horizontally by adding SFU servers across 20+ global regions."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| `max_participants_per_sfu` | 200 | 50-500 | Server resource vs. cascade complexity |
| `simulcast_layers` | 3 (720p/360p/180p) | 2-4 | Bandwidth flexibility vs. encoding CPU |
| `jitter_buffer_ms` | 50 | 20-200 | Latency vs. smoothness |
| `nack_history_ms` | 1000 | 500-3000 | Retransmission window vs. memory |
| `fec_percentage` | 10 | 0-50 | Redundancy bandwidth vs. loss resilience |
| `active_speaker_threshold_db` | -40 | -60 to -20 | Speaker detection sensitivity |
| `cascade_threshold` | 50 participants or 2+ regions | - | When to enable cascading |
| `turn_allocation_ttl_sec` | 600 | 300-3600 | TURN resource lifetime |
| `recording_composite_fps` | 24 | 15-30 | Recording quality vs. processing cost |
| `max_meeting_duration_hrs` | 24 | 1-72 | Resource reservation limit |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| **SFU** | Selective Forwarding Unit -- routes media packets without transcoding |
| **MCU** | Multipoint Control Unit -- decodes, mixes, and re-encodes media streams |
| **Simulcast** | Sending multiple quality layers of the same video simultaneously |
| **SVC** | Scalable Video Coding -- single bitstream with extractable quality layers |
| **ICE** | Interactive Connectivity Establishment -- NAT traversal framework |
| **STUN** | Protocol to discover public IP address through NAT |
| **TURN** | Relay protocol for when direct connectivity fails |
| **SRTP** | Secure Real-time Transport Protocol -- encrypted media transport |
| **DTLS** | Datagram TLS -- encryption for UDP-based protocols |
| **SDP** | Session Description Protocol -- describes media session parameters |
| **TWcc** | Transport-Wide Congestion Control -- bandwidth estimation mechanism |
| **MOS** | Mean Opinion Score -- 1-5 scale rating perceived audio/video quality |
| **Jitter buffer** | Buffer that smooths out variable packet arrival times |

## Appendix C: References

- WebRTC standard (W3C and IETF RFCs 8825-8835)
- RFC 3550 -- RTP: Real-Time Transport Protocol
- Zoom Architecture Blog: "How Zoom Provides Industry-Leading Video Quality"
- Twilio Engineering: "Scaling WebRTC Infrastructure"
- Jitsi (open-source SFU) architecture documentation
- Google Congestion Control (GCC) algorithm (draft-ietf-rmcat-gcc)
