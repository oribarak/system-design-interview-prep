# Netflix -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need to design a subscription streaming service that delivers a curated catalog of movies and shows to 250M subscribers worldwide with cinema-quality playback. Unlike YouTube, we have a known catalog, so the focus is on per-shot encoding optimization, pre-positioning content on ISP-embedded CDN appliances, and highly personalized recommendations.

## 2. Requirements
**Functional:** Encode master files into optimized renditions. Adaptive bitrate streaming with DRM (Widevine, FairPlay). Personalized home page with recommendation rows. Multi-profile accounts. Cross-device resume ("continue watching"). Download for offline viewing.

**Non-functional:** 99.99% playback availability. Playback start under 2 seconds. Rebuffer rate under 0.5%. Support 30M concurrent streams at peak (150 Tbps). Per-shot encoding to minimize bandwidth at target quality. DRM compliance per studio contracts.

## 3. Core Concept: Per-Shot Encoding with Open Connect
Netflix splits each video into shots (scene cuts), analyzes the complexity of each shot, and encodes them at individually optimized bitrate-resolution points using convex hull analysis. Simple shots (talking heads) get low bitrate; complex shots (action sequences) get high bitrate. This saves 20-50% bandwidth compared to fixed-ladder encoding. The encoded content is then pre-positioned on ISP-embedded appliances (Open Connect) during off-peak hours, achieving 95%+ cache hit rates.

## 4. High-Level Architecture
```
Mezzanine --> Encoding Pipeline (per-shot) --> S3 Origin
                                                   |
                                          Open Connect Fill (off-peak)
                                                   |
Viewers <-- ISP OCA (95%) --> IXP OCA (4%) --> S3 Origin (<1%)

Client --> API Gateway --> Recommendation / Playback / Search Services
```
- **Encoding Pipeline**: Per-shot optimization across multiple codecs (H.264, VP9, AV1).
- **Open Connect (OCAs)**: Custom hardware inside ISP networks pre-loaded with predicted popular content.
- **Steering Service**: Selects the best OCA for each session based on location, load, and content availability.
- **Recommendation Service**: Hybrid ML engine driving personalized rows and even personalized artwork.

## 5. How Playback Works
1. User selects a title on the home page (driven by recommendation engine).
2. Playback Service creates a session and queries the Steering Service for the best OCA.
3. A signed, time-limited manifest URL and DRM license are issued to the client.
4. Client fetches the DASH/HLS manifest and begins streaming segments from the ISP-embedded OCA.
5. Adaptive bitrate algorithm switches quality based on measured bandwidth.
6. Client sends periodic heartbeats (position, bitrate, rebuffers) for quality monitoring.
7. Position is saved to Cassandra for cross-device resume.

## 6. What Happens When Things Fail?
- **OCA failure**: Steering Service detects unhealthy appliances via health checks and redirects clients to the next available OCA within the same ISP or region.
- **Microservice failure (recommendations, metadata)**: Circuit breakers activate degraded mode -- show cached/precomputed recommendations. Playback continues since manifest URLs are already issued.
- **Regional AWS outage**: DNS-based failover to secondary region. OCAs continue serving cached content independently of the control plane.

## 7. Scaling
- **10x**: Deploy higher-density OCAs (1 PB each), expand to more ISPs. Encode entire catalog in AV1 for 30%+ bandwidth savings. ML-based OCA selection considering real-time congestion.
- **100x**: Predictive pre-download of likely-to-be-watched content on user devices. Neural compression codecs. Federated recommendation models trained on-device for privacy.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|---|---|
| Custom CDN (Open Connect) vs. third-party | 10x+ cost savings at Netflix scale with tighter quality control, but requires managing hardware in 1000+ ISP locations |
| Per-shot encoding vs. fixed bitrate ladder | 20-50% bandwidth savings, but requires ~1000 GPU-hours per title and is only justified for a known, watched-many-times catalog |
| Cassandra for watch history vs. DynamoDB | Proven at Netflix scale with in-house operational expertise, but requires dedicated team for cluster management |

## 9. Closing (30s)
> We designed a Netflix-scale platform serving 250M subscribers with 30M concurrent streams. Per-shot encoding reduces bandwidth 20-50% at target quality. Content is pre-positioned on ISP-embedded Open Connect appliances during off-peak hours, achieving 95%+ cache hit rates at 150 Tbps peak. Personalized recommendations drive the home feed, and playback orchestration handles DRM, CDN steering, and cross-device resume. The system is hardened with chaos engineering and A/B tests every change.
