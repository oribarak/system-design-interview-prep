# Zoom / Video Conferencing -- Cheat Sheet

## Key Numbers
- 1M concurrent meetings, 5M concurrent participants, 10K SFU servers
- Video: 720p/360p/180p simulcast at 1.5Mbps/500Kbps/150Kbps per layer
- Audio: Opus codec at 48 Kbps, 20ms frames
- p95 latency: audio < 150ms, video < 200ms glass-to-glass
- Total bandwidth: ~31 Tbps downstream, ~7.75 Tbps upstream
- Recording: ~225 TB/hour across all meetings

## Core Components (one line each)
- **Signaling Server**: Stateless WebSocket servers for SDP/ICE exchange and meeting control
- **SFU (Selective Forwarding Unit)**: Routes media packets without transcoding, selects simulcast layer per receiver
- **Media Router**: Assigns meetings to SFU clusters based on geography and load
- **TURN Server**: UDP relay for users behind restrictive NATs/firewalls (~15% of users)
- **Bandwidth Estimator (TWcc/GCC)**: Estimates per-receiver available bandwidth for layer selection
- **Recording Service**: Headless participant that captures streams and composites to MP4 post-meeting
- **Meeting Service**: CRUD, authentication, waiting room, host controls

## Architecture in One Sentence
Clients negotiate sessions via WebRTC signaling, then stream simulcast audio/video over SRTP/UDP to geographically-near SFU servers that selectively forward the appropriate quality layer to each receiver based on bandwidth estimation, with cascading across regions for distributed meetings.

## Top 3 Trade-offs
1. **SFU vs MCU**: SFU chosen for no-transcoding cost savings and lower latency, despite O(N) downstream per participant
2. **Simulcast vs SVC**: Simulcast is simpler and widely supported; SVC offers finer granularity but limited codec/browser support
3. **UDP vs TCP for media**: UDP chosen for no head-of-line blocking; TCP fallback for extreme firewall environments at +50ms latency

## Scaling Story
- **10x (50M participants)**: 100K SFU servers, more regional PoPs, 4th simulcast layer (1080p), aggressive mobile downscaling
- **100x (500M participants)**: Edge SFUs at ISP level, AV1/SVC codecs for 30-50% bandwidth savings, 3-tier cascade, own backbone network

## Closing Statement
An SFU-based architecture with 3-layer simulcast and TWcc bandwidth estimation delivers sub-200ms interactive video for 5M concurrent participants, scaling via regional SFU cascading and graceful quality degradation from 720p down to audio-only.
