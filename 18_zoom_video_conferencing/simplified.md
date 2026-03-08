# Video Conferencing (Zoom) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a video conferencing system supporting real-time audio/video for meetings from 1-on-1 to 1,000 participants. The hard part is achieving sub-200ms glass-to-glass latency while adapting video quality to each participant's network conditions and scaling media routing infrastructure globally.

## 2. Requirements
**Functional:** Join/leave meetings via link, real-time bidirectional audio/video, screen sharing, in-meeting chat, cloud recording with playback, host controls (mute, kick, waiting room).

**Non-functional:** p95 audio latency under 150ms, p95 video latency under 200ms, 1M concurrent meetings with 5M participants, adaptive quality degradation on poor networks, global media server presence in 20+ regions.

## 3. Core Concept: SFU with Simulcast
The key insight is using a Selective Forwarding Unit (SFU) instead of a Multipoint Control Unit (MCU). The SFU receives each participant's media and selectively forwards it to others without transcoding -- saving enormous CPU. Each sender publishes 3 simultaneous video layers (720p, 360p, 180p) via simulcast. The SFU picks which layer to forward to each receiver based on their bandwidth, UI layout (active speaker gets high quality, thumbnails get low), and congestion feedback.

## 4. High-Level Architecture
```
Client A --[SRTP/UDP]--> SFU Server --[SRTP/UDP]--> Client B
   |                        |                          |
   +--[WebSocket]---> Signaling Server <--[WebSocket]--+
                           |
                     [Meeting Service]
                       (Postgres)
```
- **Signaling Server**: handles WebSocket connections for SDP offer/answer exchange and meeting controls.
- **SFU (Selective Forwarding Unit)**: core media routing -- receives streams, selectively forwards to others. No transcoding.
- **TURN Server**: relay fallback for participants behind restrictive NATs/firewalls (~15% of users need it).
- **Media Router**: assigns meetings to SFU clusters by participant geography and load.
- **Recording Service**: joins as a headless participant, captures all streams, composites to MP4 post-meeting.

## 5. How Joining a Meeting Works
1. Client calls the Join API, gets a signaling WebSocket URL and ICE/TURN server details.
2. Client connects to the Signaling Server and sends a JOIN_ROOM message.
3. Client creates an SDP offer describing its media capabilities and sends it to the SFU via the signaling server.
4. SFU generates an SDP answer; both sides exchange ICE candidates for NAT traversal.
5. DTLS handshake establishes encryption keys; SRTP media begins flowing over UDP.
6. SFU receives the client's 3 simulcast layers and selectively forwards appropriate layers to each other participant.
7. SFU uses TWcc (Transport-Wide Congestion Control) feedback to estimate each receiver's bandwidth and switch layers accordingly.

## 6. What Happens When Things Fail?
- **SFU server crash**: Clients detect loss via ICE keepalives. A warm standby SFU (receiving a low-bitrate copy) takes over; clients re-establish media within 2-3 seconds.
- **Network degrades for a participant**: SFU downgrades from high to low simulcast layer, then reduces frame rate, then drops to audio-only.
- **Signaling server crash**: Stateless servers behind a load balancer; clients reconnect to another instance. Meeting state lives in Redis.

## 7. Scaling
- **10x**: Scale to 100K SFU servers across more regional PoPs. Automate provisioning with Kubernetes. Add a 4th simulcast layer (1080p) for high-bandwidth users.
- **100x**: Deploy lightweight SFUs at ISP edge locations. Adopt AV1/SVC codecs for 30-50% bandwidth savings. Use 3-tier SFU cascading (edge to regional to backbone) for inter-region traffic.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| SFU over MCU | No transcoding CPU cost, but downstream bandwidth is O(N) per participant |
| UDP over TCP for media | No head-of-line blocking, but unreliable -- need NACK/FEC for packet loss recovery |
| Simulcast (3 layers) | Adaptive quality per receiver, but sender uses 3x encoding CPU and upstream bandwidth |

## 9. Closing (30s)
> We designed a video conferencing system using SFU-based media routing with 3-layer simulcast for adaptive quality. Clients negotiate sessions via WebRTC signaling, then stream audio/video over encrypted UDP to geographically-near SFU servers. The SFU selects which quality layer to forward to each receiver based on bandwidth estimates from TWcc feedback. For large or geo-distributed meetings, SFUs cascade across regions. The system degrades gracefully from 720p to 360p to audio-only as conditions worsen.
