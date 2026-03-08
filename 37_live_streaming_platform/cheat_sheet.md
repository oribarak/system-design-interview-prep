# Live Streaming Platform -- Cheat Sheet

## Key Numbers
- 100K concurrent live channels, 30M concurrent viewers
- Max 5M viewers per stream, peak egress 120 Tbps
- Ingest bitrate: 6 Mbps per broadcaster (1080p60)
- Segment duration: 2s (standard), 0.5s partials (LL-HLS)
- Glass-to-glass latency: < 5s (standard), < 1s (WebRTC)
- Chat: 500K msg/sec platform-wide, 50K msg/sec per top channel
- VOD storage: ~600 TB/day

## Core Components
- **Ingest Servers**: Geo-distributed RTMP/SRT receivers with stream key auth
- **GPU Transcoders**: Dedicated per-stream, 4 ABR renditions, 2s HLS segments
- **Origin Servers**: Segment storage + playlist updates
- **CDN Edge PoPs**: Cache + serve HLS to viewers (~200 locations)
- **WebSocket Gateway + Redis Streams**: Live chat pub/sub with batching/sampling
- **VOD Recorder**: Captures live segments to S3 for on-demand replay
- **AutoMod**: ML-based real-time chat moderation

## Architecture in One Sentence
Broadcasters send RTMP to geo-distributed ingest servers feeding dedicated GPU transcoders that produce 2-second HLS segments, fanned out through a multi-tier CDN to millions of viewers, with live chat delivered via WebSocket gateways backed by Redis Streams using server-side batching.

## Top 3 Trade-offs
1. **HLS vs. WebRTC delivery**: HLS leverages CDN caching (scales to millions) but adds 3-5s latency; WebRTC is sub-second but cannot use CDN
2. **Dedicated vs. shared transcoders**: Dedicated avoids noisy-neighbor but wastes resources on small streams
3. **Chat batching/sampling vs. full delivery**: Essential for 5M-viewer channels but reduces chat fidelity

## Scaling Story
- 10x: 10x GPU fleet, multi-CDN steering, per-channel Redis sharding for chat
- 100x: P2P-assisted delivery (WebRTC mesh), edge transcoding, regionalized chat rooms

## Closing Statement
A live streaming platform with geo-distributed RTMP ingest, dedicated GPU transcoders producing multi-rendition HLS, CDN fan-out to 5M+ viewers per stream with < 5s latency, and Redis Streams-backed live chat with batching and ML moderation -- handling 100K concurrent broadcasts and 30M total viewers.
