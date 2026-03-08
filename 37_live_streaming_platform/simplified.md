# Live Streaming Platform -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to build a live streaming platform where creators broadcast video to potentially millions of concurrent viewers in real time, with interactive chat. The hard parts are achieving under-5-second glass-to-glass latency, scaling a single stream to 5M viewers through CDN fan-out, and handling 50K chat messages per second in a top channel.

## 2. Requirements
**Functional:** Accept RTMP/SRT ingest from broadcaster software. Real-time transcode into 4 ABR renditions. Deliver HLS segments to viewers via CDN. Live chat with moderation. Record streams for VOD replay.

**Non-functional:** Glass-to-glass latency under 5 seconds (standard) or under 1 second (low-latency mode). Scale a single stream to 5M viewers. 30M total concurrent viewers. 99.99% delivery availability. Chat throughput of 500K messages/sec platform-wide.

## 3. Core Concept: CDN Fan-Out with Segment Caching
A single broadcaster's stream is transcoded into HLS segments (2 seconds each) by a dedicated GPU transcoder. These segments are pushed to an origin server that fans out through a CDN to hundreds of edge PoPs worldwide. Each PoP caches segments and serves thousands of viewers locally. The CDN absorbs the 1-to-millions fan-out problem -- 200 PoPs serving 25K viewers each makes 5M total viewers manageable. For ultra-low latency, LL-HLS uses partial segments (0.5s) with blocking playlist requests.

## 4. High-Level Architecture
```
Broadcaster (RTMP) --> Ingest Server --> Live Transcoder (GPU)
                                              |
                                         Origin Server
                                              |
                                    CDN Edge PoPs (HLS)
                                              |
                                           Viewers

Chat: Viewers <-> WebSocket Gateway <-> Chat Service <-> Redis Streams
```
- **Ingest Servers**: Geo-distributed, authenticate via stream key, forward to transcoders.
- **Live Transcoder**: Dedicated GPU per stream producing 4 renditions as 2-second HLS segments.
- **CDN Edge PoPs**: Cache segments, handle viewer fan-out, stagger polling with jitter.
- **Chat Service**: WebSocket gateways backed by Redis Streams for per-channel pub/sub.

## 5. How a Live Stream Reaches Viewers
1. Broadcaster connects via RTMP to the nearest ingest server using their stream key.
2. Ingest server authenticates and forwards the stream to a dedicated GPU transcoder.
3. Transcoder produces 4 renditions (1080p, 720p, 480p, 360p) as 2-second HLS segments.
4. Segments are pushed to the origin server, which updates the HLS playlist.
5. CDN edge PoPs pull new segments (or receive proactive pushes for popular streams).
6. Viewers' players poll for playlist updates every 2 seconds and download new segments.
7. Total latency: ~2s encoding + 2s segment + 1s propagation = ~5 seconds.

## 6. What Happens When Things Fail?
- **Transcoder failure**: Health monitor detects within 2 seconds. A standby transcoder takes over using the backup ingest feed. Viewers experience a brief 2-3 second freeze.
- **CDN edge PoP failure**: DNS routes viewers to the next nearest PoP. Existing connections fail over at the HTTP level on the next segment request.
- **Chat service failure**: WebSocket gateway auto-reconnects viewers. Redis Streams replicas prevent message loss. Users see a brief "reconnecting" indicator.

## 7. Scaling
- **10x**: Scale GPU transcoder fleet with spot instances for smaller streams. Multi-CDN strategy with real-time quality-based steering. Dedicated Redis clusters for top 100 channels.
- **100x**: P2P-assisted delivery where each viewer relays segments to 2-3 nearby peers. Edge transcoding at CDN PoPs. Regionalized chat rooms to reduce fan-out.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| HLS delivery (CDN-cacheable) vs. WebRTC (sub-second) | HLS scales to millions via caching but has ~5s latency; WebRTC is sub-second but loses CDN caching benefits |
| Dedicated GPU transcoder per stream | Avoids noisy-neighbor quality issues, but less cost-efficient for small streams than a shared pool |
| Chat batching + sampling at high volume | Maintains viewer experience at scale, but some messages are never displayed in very large channels |

## 9. Closing (30s)
> We designed a live platform handling 100K concurrent broadcasts and 30M viewers. Broadcasters send RTMP to geo-distributed ingest servers feeding dedicated GPU transcoders that produce HLS segments. CDN fan-out scales a single stream to 5M viewers with under 5 seconds latency. Live chat uses WebSocket gateways and Redis Streams with batching and sampling for top channels. Streams are recorded for VOD with time-synced chat replay.
