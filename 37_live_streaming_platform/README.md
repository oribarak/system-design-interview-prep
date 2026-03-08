# System Design Interview: Live Streaming Platform (like Twitch) -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a live streaming platform -- similar to Twitch -- where creators broadcast live video to potentially millions of concurrent viewers in real-time, with interactive features like live chat. The core challenges are ultra-low latency ingest-to-viewer delivery (under 5 seconds), scaling a single stream to millions of viewers via CDN, handling live chat at high message throughput, and recording/transcoding live streams for VOD replay. I will cover the ingest pipeline, transcoding, CDN fan-out, live chat, and VOD recording."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we designing the core live video pipeline, or also features like clips, raids, subscriptions, and donations?
- Should the platform support co-streaming (multiple streamers in one session)?
- Do we need to support both RTMP ingest (OBS) and browser-based streaming (WebRTC)?

**Scale**
- How many concurrent live channels? (Assume 100,000 simultaneous live streams.)
- How many concurrent viewers on the largest stream? (Assume 5 million for a major event.)
- Total concurrent viewers platform-wide? (Assume 30 million.)

**Policy / Constraints**
- What is the target glass-to-glass latency? (Assume < 5 seconds for standard, < 1 second for interactive mode.)
- Do we need live content moderation? (Yes -- automated for TOS violations.)
- Should we record all streams for VOD? (Yes, with streamer opt-in.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Concurrent live channels | 100,000 |
| Total concurrent viewers | 30,000,000 |
| Max viewers per channel | 5,000,000 |
| Avg ingest bitrate per streamer | 6 Mbps (1080p60) |
| Total ingest bandwidth | 100K x 6 Mbps = 600 Gbps |
| Transcoded renditions per stream | 4 (1080p, 720p, 480p, 360p) |
| Total transcode output | 100K x 4 x ~3 Mbps avg = 1.2 Tbps |
| Avg viewer bitrate | 4 Mbps |
| Peak viewer egress | 30M x 4 Mbps = 120 Tbps |
| Chat messages per second (platform) | ~500,000 |
| Chat messages per second (top channel) | ~50,000 |
| VOD storage per day | 100K channels x avg 3 hrs x 2 GB/hr = 600 TB/day |

---

## 3) Requirements (3-5 min)

### Functional
- **Live ingest**: Accept RTMP/SRT streams from broadcaster software (OBS, Streamlabs).
- **Live transcoding**: Real-time transcode into multiple ABR renditions (1080p, 720p, 480p, 360p).
- **Live delivery**: HLS/DASH segments delivered to viewers via CDN with < 5s latency.
- **Live chat**: Real-time chat per channel with emotes, moderation, and slow mode.
- **VOD recording**: Record live streams and make them available as on-demand videos.
- **Channel discovery**: Browse by category/game, recommended channels, search.

### Non-functional
- **Latency**: Glass-to-glass < 5 seconds (standard); < 1 second (low-latency mode via WebRTC/LL-HLS).
- **Scale**: Single stream to 5M viewers; 30M total concurrent viewers.
- **Availability**: 99.99% for live video delivery during streams.
- **Reliability**: Zero dropped frames during stable network conditions.
- **Chat throughput**: 500K messages/sec platform-wide, 50K/sec per top channel.
- **VOD availability**: Stream available as VOD within 5 minutes of broadcast end.

---

## 4) API Design (2-4 min)

### Get Stream Key
```
GET /api/v1/channels/{channel_id}/stream-key
Headers: Authorization: Bearer <token>

Response: { stream_key: "live_EXAMPLE_KEY", ingest_url: "rtmp://ingest.example.com/live" }
```

### Start Watching (Get Playback Info)
```
GET /api/v1/channels/{channel_id}/playback

Response: {
  status: "live",
  manifest_url: "https://cdn.example.com/live/{channel_id}/master.m3u8",
  low_latency_url: "wss://ll.example.com/live/{channel_id}",
  viewer_count: 1234567,
  started_at: "2026-03-07T14:00:00Z"
}
```

### Send Chat Message
```
POST /api/v1/channels/{channel_id}/chat
Body: { message: "GG! PogChamp", emotes: [{ id: "pogchamp", positions: [4, 12] }] }

Response: { message_id: "msg_789", timestamp: "..." }
```

### Receive Chat (WebSocket)
```
WS /api/v1/channels/{channel_id}/chat/ws

Server pushes: { type: "message", user: "viewer42", message: "GG!", badges: ["subscriber"], color: "#FF0000" }
```

### Get VOD
```
GET /api/v1/vods/{vod_id}

Response: { vod_id, channel_id, title, duration_sec, manifest_url, recorded_at, view_count }
```

---

## 5) Data Model (3-5 min)

### Channels Table
| Column | Type | Notes |
|---|---|---|
| channel_id | STRING (PK) | |
| user_id | STRING (FK) | Owner |
| name | STRING | Display name |
| stream_key | STRING (UNIQUE) | Secret key for ingest auth |
| category | STRING | Game/category |
| is_live | BOOLEAN | Currently streaming |
| current_viewer_count | INT | Real-time (approximate) |
| follower_count | BIGINT | |
| created_at | TIMESTAMP | |

### Streams Table (per broadcast session)
| Column | Type | Notes |
|---|---|---|
| stream_id | STRING (PK) | |
| channel_id | STRING (FK) | |
| started_at | TIMESTAMP | |
| ended_at | TIMESTAMP | Null if live |
| peak_viewers | INT | |
| avg_viewers | INT | |
| vod_path | STRING | S3 path for recorded VOD |
| status | ENUM | LIVE, ENDED, PROCESSING_VOD |

### Chat Messages (Hot Storage)
| Column | Type | Notes |
|---|---|---|
| channel_id | STRING (PK) | Partition key |
| timestamp | TIMESTAMP (SK) | Sort key |
| message_id | STRING | |
| user_id | STRING | |
| message | STRING | Max 500 chars |
| emotes | JSON | Emote metadata |
| is_deleted | BOOLEAN | Moderation flag |

**Storage choices**: Channel/stream metadata in sharded PostgreSQL. Chat in Redis Streams (hot, last 24h) with archival to Cassandra. Viewer counts in Redis (approximate, updated every 5s). Video segments in S3 with CDN origin. VOD manifests in S3.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Broadcaster** sends RTMP/SRT stream to the nearest **Ingest Server**.
2. **Ingest Server** authenticates via stream key, demuxes the stream, and forwards to the **Transcoder**.
3. **Transcoder** encodes into 4 ABR renditions in real-time and produces 2-second HLS segments.
4. Segments are pushed to the **Origin** server, which distributes to **CDN Edge PoPs**.
5. **Viewers** connect to the nearest edge PoP and receive HLS segments with < 5s glass-to-glass latency.
6. For low-latency mode, viewers use **LL-HLS** (partial segments) or **WebRTC** (sub-second).
7. **Chat Service** manages WebSocket connections for live chat; messages are published to channel-specific **Redis Streams** and fan-out to connected viewers.
8. **VOD Recorder** captures segments during the live broadcast and assembles them into a complete VOD file post-stream.
9. **Moderation Service** scans chat messages and video frames for TOS violations.

### Components
- **Ingest Servers**: Accept RTMP/SRT, authenticate, forward to transcoders. Geo-distributed.
- **Live Transcoders**: Real-time GPU encoding into multiple renditions. Stateful (one transcoder per stream).
- **Origin Servers**: Receive transcoded segments, serve to CDN. Fan-out point.
- **CDN Edge PoPs**: Cache and serve HLS segments to viewers. Handle viewer fan-out.
- **Chat Service**: WebSocket gateway + Redis Streams for pub/sub per channel.
- **Chat Moderation**: ML + rule-based filtering (banned words, spam detection).
- **VOD Recording Service**: Writes segments to S3, assembles VOD post-stream.
- **Channel Discovery Service**: Browse, search, and recommendation for live channels.
- **Viewer Count Service**: Approximate real-time viewer counts via Redis.
- **Clip Service**: Viewers create short clips from live moments.

> **One-sentence summary**: Broadcasters send RTMP streams to geo-distributed ingest servers that feed real-time GPU transcoders producing HLS segments, which fan out through a CDN to millions of viewers, while live chat flows through WebSocket gateways backed by Redis Streams pub/sub.

---

## 7) Deep Dive #1: Live Video Pipeline -- Ingest to Viewer (8-12 min)

The core challenge is delivering live video from one broadcaster to potentially 5 million viewers with under 5 seconds of latency.

### Ingest
- **Protocol**: RTMP (most common from OBS) or SRT (newer, better for lossy networks).
- **Geo-distributed ingest PoPs**: 20+ locations worldwide. The broadcaster connects to the nearest ingest server (DNS-based routing).
- **Authentication**: Stream key validated against the database. Invalid keys are rejected immediately.
- **Redundancy**: The broadcaster can send to two ingest servers simultaneously (primary + backup). The system selects the best-quality feed.

### Live Transcoding
- **Dedicated transcoder per stream**: Each live stream is assigned to one transcoder instance (GPU-accelerated, NVENC or similar).
- **Rendition ladder**: 1080p60 (source), 720p30 (2.5 Mbps), 480p30 (1 Mbps), 360p30 (600 Kbps).
- **Segment duration**: 2 seconds for standard latency; partial segments (0.5s) for LL-HLS.
- **Keyframe alignment**: Force keyframes at segment boundaries for clean ABR switching.
- **Transcoder failover**: A health monitor detects transcoder failures. A standby transcoder takes over within 2-3 seconds using the backup ingest feed.

### Segment Distribution (Origin -> CDN)
1. Transcoder pushes completed segments to the **Origin** server.
2. Origin updates the HLS playlist (`.m3u8`) to include the new segment.
3. CDN edge servers poll the origin for new segments (pull) or receive push notifications.
4. **Push optimization**: For popular streams (>100K viewers), origin pushes segments proactively to top-tier edge PoPs to reduce latency.

### Viewer Playback
- Standard HLS: Player polls for playlist updates every segment duration (2s). Total latency: 2s (encoding) + 2s (segment) + 1s (CDN propagation) = ~5s.
- **LL-HLS**: Partial segments (0.5s) with blocking playlist requests. Player sends a request that blocks until the next partial segment is ready. Latency: ~2-3s.
- **WebRTC (ultra-low)**: For interactive use cases (auctions, gaming). Sub-second latency but requires more server resources (no CDN caching benefit for WebRTC; use SFU architecture).

### Scaling a Single Stream to 5M Viewers
- A single origin server cannot serve 5M viewers directly. The CDN absorbs the fan-out.
- Edge PoPs cache segments. 200 PoPs with ~25K viewers each = manageable.
- **Thundering herd**: When a new segment is published, all viewers request it simultaneously. Mitigation: stagger playlist polling with jitter; edge PoPs coalesce cache-miss requests.
- **Multicast within PoPs**: Some edge architectures use internal multicast to serve many viewers from a single segment copy in memory.

---

## 8) Deep Dive #2: Live Chat at Scale (5-8 min)

Chat is integral to the live streaming experience. The top channels see 50,000 messages per second.

### Architecture
```
Viewer <-> WebSocket Gateway <-> Chat Service <-> Redis Streams (per channel)
```

1. **WebSocket Gateway**: Horizontally scaled fleet of WebSocket servers. Each viewer maintains one persistent WebSocket connection. Consistent hashing maps channels to gateway clusters.
2. **Chat Service**: Receives messages via HTTP/gRPC from the gateway. Validates, applies moderation, and publishes to the channel's Redis Stream.
3. **Redis Streams**: Each channel has a Redis Stream key. New messages are appended. Gateway servers subscribe to the stream and fan out to connected viewers.

### Fan-out Challenge
- A channel with 5M viewers and 50K msg/sec: naively, that is 5M x 50K = 250 billion deliveries/sec. Impossible.
- **Solution: Server-side batching and sampling.**
  1. **Batch**: Aggregate messages into 100ms windows. Send one WebSocket frame with 50 messages (5K msg/sec visible to viewer, not 50K).
  2. **Sample**: For very high-volume channels, randomly sample messages to show (e.g., show 30% of messages). Subscriber/moderator messages are always shown.
  3. **Slow mode**: Streamer enables a cooldown (e.g., 3 seconds between messages per user), reducing volume at the source.

### Moderation Pipeline
- **AutoMod (ML)**: Classify messages for toxicity, hate speech, spam in real-time (< 50ms per message).
- **Ban list**: Maintain per-channel ban lists in Redis. Banned users' messages are silently dropped.
- **Rate limiting**: Per-user rate limit (20 msg/min for non-subscribers, 30 for subscribers).
- **Moderator tools**: Real-time dashboard to timeout/ban users, delete messages. Deleted messages are broadcast as removal events to connected viewers.

### Chat Persistence
- **Hot**: Redis Streams (last 24 hours). Fast reads for replay.
- **Cold**: Archived to Cassandra (keyed by channel_id, time range). Used for VOD chat replay.
- **Chat replay on VOD**: When watching a VOD, chat messages are served time-synced with the video position.

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| Ingest protocol | RTMP + SRT | WebRTC ingest | RTMP is universal (OBS support); SRT handles packet loss better. WebRTC adds complexity for most streamers |
| Delivery protocol | HLS (standard + LL-HLS) | WebRTC delivery | HLS leverages CDN caching (critical for 5M viewers). WebRTC is only for interactive/ultra-low-latency |
| Chat pub/sub | Redis Streams | Kafka | Redis has lower latency for real-time chat. Kafka is better for event durability but overkill for ephemeral chat |
| Transcoder model | Dedicated GPU per stream | Shared transcoder pool | Dedicated avoids noisy-neighbor; shared is more cost-efficient for small streams |
| Chat fan-out | Batching + sampling | Deliver every message | Delivering 50K msg/sec to 5M viewers is infeasible; batching maintains the experience |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (300M concurrent viewers, 1M live channels)
- **Transcoder fleet**: 10x GPU instances. Use spot instances for smaller streams, reserved for top streamers.
- **CDN expansion**: Partner with additional CDN providers (multi-CDN strategy). Use real-time quality metrics to steer viewers to the best-performing CDN.
- **Chat sharding**: Shard Redis by channel. Dedicated Redis clusters for top 100 channels.
- **Ingest PoPs**: Expand to 50+ locations for global coverage.

### 100x (3B concurrent viewers -- hypothetical global event)
- **P2P-assisted delivery**: WebRTC-based peer mesh among viewers supplements CDN. Each viewer relays segments to 2-3 nearby peers, reducing CDN load by 50%+.
- **Edge transcoding**: Place transcoders at CDN edge PoPs to reduce origin-to-edge latency.
- **Regionalized chat**: Split chat by viewer region (e.g., separate chat rooms per country) to reduce fan-out.
- **Preemptive CDN capacity**: For scheduled events, pre-provision CDN capacity and cache warmer.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **Ingest server failure**: Broadcaster software auto-reconnects to the next nearest ingest server (DNS failover). Backup ingest feed ensures continuity.
- **Transcoder failure**: Health monitor detects within 2 seconds. Standby transcoder starts from the ingest feed. Viewers experience a brief freeze (2-3s), then resume.
- **CDN edge failure**: Viewers are redirected to next nearest PoP via DNS. Existing connections fail over at the HTTP level (next segment request goes to a new PoP).
- **Chat service failure**: WebSocket gateway reconnects viewers automatically. Redis Streams replicas prevent message loss. Viewers see a brief "reconnecting" indicator.
- **Origin server failure**: Replicated origin servers with active-passive failover. CDN edge PoPs cache recent segments, so a short origin outage is invisible to viewers.
- **Stream quality degradation**: If the broadcaster's network degrades, the ingest server detects bitrate drops and switches to a lower ingest quality. Transcoder adjusts renditions accordingly.

---

## 12) Observability and Operations (2-4 min)

- **Stream health**: Per-stream dashboard showing ingest bitrate, frame drops, transcoder latency, segment push latency.
- **Glass-to-glass latency**: Measured by embedding timestamps in video frames and comparing at the viewer. Track p50, p95.
- **Viewer experience**: Rebuffer rate, time-to-first-frame, ABR quality distribution per PoP.
- **Chat latency**: Time from message send to delivery to other viewers. Target < 500ms.
- **CDN cache hit rate**: Per PoP, per stream tier (top 100 vs. long tail).
- **Transcoder utilization**: GPU usage per instance. Auto-scale based on live channel count.
- **Alerting**: Alert on ingest failures (stream key auth errors), transcoder backlogs, CDN error rates, chat delivery latency spikes.

---

## 13) Security (1-3 min)

- **Stream key authentication**: Unique, rotatable stream keys per channel. Rate limit failed auth attempts.
- **DRM for premium content**: Widevine/FairPlay for paid events or subscriber-only streams.
- **Chat security**: Rate limiting, AutoMod for toxicity, IP-based ban for persistent abusers. Captcha for suspicious accounts.
- **Stream recording consent**: Streamers control whether their stream is recorded. DMCA takedown process for copyrighted content.
- **TLS everywhere**: RTMPS (TLS-wrapped RTMP) for ingest. HTTPS for HLS delivery. WSS for chat.
- **Anti-piracy**: Stream watermarking for high-value content (sports, PPV events).

---

## 14) Team and Operational Considerations (1-2 min)

- **Video pipeline team**: Ingest servers, transcoders, origin servers. GPU fleet management.
- **CDN/Delivery team**: CDN configuration, multi-CDN steering, edge optimization.
- **Chat team**: WebSocket infrastructure, Redis Streams, moderation tools.
- **Discovery/Recommendations team**: Channel rankings, category browsing, personalized suggestions.
- **Trust & Safety team**: AutoMod models, content policy enforcement, ban management.
- **Deployment**: Rolling updates for stateless services. Transcoder updates require draining (wait for stream to end or migrate to new instance between keyframes).

---

## 15) Common Follow-up Questions

**Q: How does the clip feature work?**
A: Viewers click "clip" specifying a time range (e.g., last 30 seconds). The server extracts the corresponding segments from the VOD recording buffer, trims, and generates a shareable short video. Clips are stored independently in S3.

**Q: How do you handle stream sniping (cheating in competitive games)?**
A: Offer a configurable stream delay (e.g., 30-60 seconds) that buffers segments before releasing to viewers. The broadcaster sees their real-time feed; viewers are delayed.

**Q: How does co-streaming work?**
A: Each co-streamer sends their own RTMP stream. A composition service mixes video/audio into a combined layout (picture-in-picture, side-by-side). The composed stream is then transcoded and distributed normally.

**Q: How do you handle a streamer going viral mid-stream?**
A: Auto-scale CDN resources. The origin detects viewer count growth rate and proactively pushes segments to more edge PoPs. Transcoder remains stable (1 stream = 1 transcoder). Chat switches to sampled mode at high viewer counts.

**Q: How is VOD different from the live pipeline?**
A: After the stream ends, recorded segments are stitched, optionally re-encoded at higher quality (offline transcoding allows more compute per frame), and stored as standard HLS/DASH VOD. Chat messages are archived with timestamps for synced replay.

---

## 16) Closing Summary (30-60s)

> "We designed a live streaming platform handling 100K concurrent broadcasts and 30M concurrent viewers. Broadcasters send RTMP/SRT to geo-distributed ingest servers, which feed dedicated GPU transcoders producing 4 ABR renditions as 2-second HLS segments. Segments fan out through a CDN to edge PoPs worldwide, achieving < 5 seconds glass-to-glass latency (or < 1 second with LL-HLS/WebRTC). Live chat uses WebSocket gateways backed by Redis Streams with server-side batching and sampling to handle 50K messages/sec in top channels. Streams are recorded for VOD with chat replay. The system scales a single stream to 5M viewers through CDN fan-out and handles viral growth through proactive edge warming."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `segment_duration` | 2s | Lower for less latency (LL-HLS: 0.5s partials) |
| `rendition_count` | 4 | More for quality range; fewer for transcoder savings |
| `ingest_bitrate_cap` | 8 Mbps | Higher for partners; lower for new streamers |
| `chat_batch_interval` | 100ms | Lower for faster chat; higher for bandwidth savings |
| `chat_sample_rate` | 100% (30% for >100K viewers) | Adjust based on viewer count |
| `slow_mode_interval` | Off (configurable per channel) | Streamer sets 3-30s cooldown |
| `vod_recording` | Enabled | Streamer opt-in/out |
| `cdn_push_threshold` | 100K viewers | Push segments to more PoPs above this |
| `transcoder_failover_timeout` | 2s | Faster detection vs. false positives |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **RTMP** | Real-Time Messaging Protocol; the standard for live stream ingest from OBS |
| **SRT** | Secure Reliable Transport; newer protocol with better packet loss handling |
| **LL-HLS** | Low-Latency HLS; Apple's extension using partial segments and blocking playlists |
| **Glass-to-Glass** | Total latency from broadcaster's camera to viewer's screen |
| **SFU** | Selective Forwarding Unit; WebRTC server that forwards streams without re-encoding |
| **ABR** | Adaptive Bitrate; viewer switches quality based on bandwidth |
| **Ingest PoP** | Point of Presence that receives broadcaster streams |
| **VOD** | Video on Demand; recorded version of a live stream |
| **AutoMod** | Automated chat moderation using ML classifiers |
| **Thundering Herd** | Simultaneous requests from all viewers when a new segment is published |

## Appendix C: References

- Twitch Engineering Blog: Live Video Infrastructure
- Low-Latency HLS (Apple WWDC 2019)
- SRT Protocol Specification (Haivision)
- WebRTC for Live Streaming (Millicast, LiveKit)
- Redis Streams documentation and pub/sub patterns
