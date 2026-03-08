# System Design Interview: Netflix -- "Perfect Answer" Playbook

---

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a Netflix-style video streaming service that delivers a curated catalog of movies and TV shows to hundreds of millions of subscribers worldwide with cinema-quality playback. Unlike YouTube (user-generated, upload-heavy), Netflix has a known catalog, so the focus shifts to encoding optimization, global content delivery via Open Connect, highly personalized recommendations, and seamless multi-device playback with DRM. I will cover the content ingestion pipeline, encoding strategy, CDN architecture, recommendation engine, and playback orchestration."

---

## 1) Clarifying Questions (2-5 min)

**Scope**
- Are we focusing on the streaming platform, or also the content management and studio pipeline?
- Should we include the download-for-offline feature?
- Do we need to handle simultaneous streams per account (e.g., 4 screens on premium)?

**Scale**
- How many subscribers? (Assume 250 million.)
- How many concurrent streams at peak? (Assume 30 million.)
- How large is the content catalog? (Assume 15,000 titles, average 90 minutes.)

**Policy / Constraints**
- What playback quality target? (Up to 4K HDR Dolby Vision.)
- DRM requirements? (Widevine for Android/Chrome, FairPlay for Apple, PlayReady for Windows.)
- Content licensing by region? (Yes -- geo-restricted catalog.)

---

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Value |
|---|---|
| Subscribers | 250,000,000 |
| Daily active users | ~100,000,000 |
| Concurrent streams (peak) | 30,000,000 |
| Catalog size | 15,000 titles |
| Avg title duration | 90 min |
| Total content hours | 22,500 hours |
| Avg encoded size per title (all renditions) | ~100 GB |
| Total storage (all renditions) | 15K x 100 GB = 1.5 PB |
| Avg stream bitrate | 5 Mbps |
| Peak egress bandwidth | 30M x 5 Mbps = 150 Tbps |
| API requests (metadata, search, browse) | ~200,000 QPS |
| New title additions per week | ~50 |
| Encoding compute per title | ~1,000 GPU-hours |

---

## 3) Requirements (3-5 min)

### Functional
- **Content ingestion**: Accept master-quality files (mezzanine), encode into optimized renditions.
- **Streaming playback**: Adaptive bitrate streaming with DRM across web, mobile, TV, and console.
- **Personalized home page**: Rows of recommendations ("Because you watched...", "Trending", genre rows).
- **Search and browse**: Search by title, actor, genre; browse curated collections.
- **User profiles**: Multiple profiles per account with independent watch history and preferences.
- **Download for offline**: Select titles for offline viewing with DRM-protected local storage.
- **Continue watching**: Resume playback at the exact position across devices.

### Non-functional
- **Availability**: 99.99% for playback.
- **Playback start**: < 2 seconds.
- **Rebuffer rate**: < 0.5% of playback time.
- **Global coverage**: Low-latency streaming in 190+ countries.
- **DRM compliance**: Content must be protected per studio contracts.
- **Cost efficiency**: Optimize encoding to minimize CDN bandwidth at target quality.

---

## 4) API Design (2-4 min)

### Browse / Home Feed
```
GET /api/v1/home?profile_id={id}&country={cc}

Response: {
  rows: [
    { title: "Continue Watching", items: [{ title_id, name, thumbnail, progress_pct }] },
    { title: "Because you watched Breaking Bad", items: [...] },
    { title: "Trending Now", items: [...] }
  ]
}
```

### Get Title Details
```
GET /api/v1/titles/{title_id}?profile_id={id}

Response: {
  title_id, name, synopsis, year, rating, genres, cast,
  seasons: [{ season_num, episodes: [{ episode_id, name, duration }] }],
  maturity_rating, available_until
}
```

### Start Playback Session
```
POST /api/v1/playback/start
Body: { profile_id, title_id, episode_id?, device_type, drm_system }

Response: {
  playback_session_id, manifest_url (signed, time-limited),
  license_url, resume_position_sec,
  cdn_urls: [{ server: "oc-edge-1.isp.com", priority: 1 }]
}
```

### Report Playback Position
```
POST /api/v1/playback/heartbeat
Body: { playback_session_id, position_sec, buffer_health_sec, bitrate_kbps, rebuffer_count }
```

### Search
```
GET /api/v1/search?q={query}&profile_id={id}&country={cc}

Response: { results: [{ title_id, name, type, thumbnail, match_score }] }
```

---

## 5) Data Model (3-5 min)

### Titles Table
| Column | Type | Notes |
|---|---|---|
| title_id | STRING (PK) | UUID |
| name | STRING | Localized per region |
| type | ENUM | MOVIE, SERIES |
| synopsis | TEXT | |
| year | INT | Release year |
| genres | STRING[] | |
| maturity_rating | STRING | TV-MA, PG-13, etc. |
| duration_sec | INT | For movies; null for series |
| available_regions | STRING[] | Geo-restriction list |
| encoding_status | ENUM | PENDING, ENCODING, READY |
| created_at | TIMESTAMP | |

### Episodes Table
| Column | Type | Notes |
|---|---|---|
| episode_id | STRING (PK) | |
| title_id | STRING (FK) | Parent series |
| season | INT | |
| episode_num | INT | |
| name | STRING | |
| duration_sec | INT | |
| manifest_path | STRING | S3 path to DASH/HLS manifest |

### User Profiles Table
| Column | Type | Notes |
|---|---|---|
| profile_id | STRING (PK) | |
| account_id | STRING (FK) | Parent account |
| name | STRING | |
| avatar_url | STRING | |
| maturity_level | STRING | |
| language | STRING | |

### Watch History
| Column | Type | Notes |
|---|---|---|
| profile_id | STRING (PK) | Partition key |
| title_id | STRING (SK) | Sort key |
| episode_id | STRING | Nullable for movies |
| last_position_sec | INT | Resume position |
| watch_pct | FLOAT | Completion signal |
| last_watched | TIMESTAMP | |

### Viewing Activity (for recommendations)
| Column | Type | Notes |
|---|---|---|
| profile_id | STRING | Partition key |
| event_time | TIMESTAMP | Sort key |
| title_id | STRING | |
| action | ENUM | PLAY, PAUSE, SEEK, COMPLETE, THUMBS_UP, THUMBS_DOWN |
| device_type | STRING | |

**Storage choices**: Titles/episodes in a sharded PostgreSQL cluster (small catalog, complex queries). Watch history in Cassandra (high-write, wide rows per profile). Viewing activity in Kafka -> data warehouse for ML training. Manifests and video segments in S3 + Open Connect appliances.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. **Content team** uploads mezzanine files to the **Encoding Pipeline**, which produces optimized renditions stored in **S3 Origin**.
2. **Open Connect CDN** pre-positions popular content on **ISP-embedded appliances** during off-peak hours. Less popular content is served from **regional PoPs** or **S3 origin**.
3. **Client device** requests the **Home Feed** from the **Recommendation Service**, which returns personalized rows.
4. When user selects a title, the **Playback Service** creates a session, selects the optimal CDN server (via **Steering Service**), generates a signed manifest URL, and issues a DRM license.
5. The client fetches the **manifest** and begins streaming segments from the selected Open Connect appliance using adaptive bitrate.
6. The client periodically sends **heartbeat** events (position, bitrate, rebuffers) to the Playback Service.
7. **Event Pipeline** (Kafka + Flink) processes viewing events in real-time for continue-watching updates, and in batch for recommendation model training.

### Components
- **Encoding Pipeline**: Encodes mezzanine files into optimized renditions with per-shot encoding.
- **S3 Origin**: Stores all encoded segments and manifests.
- **Open Connect (CDN)**: Netflix's custom CDN; appliances deployed inside ISPs.
- **Steering Service**: Selects the optimal CDN server for each playback session.
- **API Gateway (Zuul)**: Authentication, routing, rate limiting.
- **Recommendation Service**: ML-based personalization for the home feed.
- **Playback Service**: Session management, DRM license issuance, resume position.
- **Metadata Service**: Title catalog, episode info, localized metadata.
- **Search Service**: Elasticsearch-backed search with ranking.
- **Event Pipeline (Kafka + Flink)**: Real-time and batch processing of viewing events.
- **Data Warehouse**: Training data for recommendation models.
- **A/B Testing Platform**: Tests new recommendation algorithms, UI changes, encoding profiles.

> **One-sentence summary**: A curated catalog is encoded with per-shot optimization and pre-positioned on ISP-embedded CDN appliances, while a personalized recommendation engine drives the home feed and a playback service orchestrates DRM-protected adaptive bitrate streaming with real-time quality monitoring.

---

## 7) Deep Dive #1: Video Encoding and Per-Shot Optimization (8-12 min)

Netflix's encoding strategy is fundamentally different from YouTube's. With a known, relatively small catalog (~15K titles), Netflix invests heavily in encoding quality to minimize bandwidth while maximizing perceptual quality.

### Per-Shot Encoding
Traditional encoding uses a fixed bitrate ladder for all content. Netflix's approach:

1. **Shot Detection**: Split the video into individual shots (scene cuts). A 90-min movie has ~2,000 shots.
2. **Complexity Analysis**: For each shot, measure spatial complexity (detail) and temporal complexity (motion). A talking-head shot is simple; an action sequence is complex.
3. **Convex Hull Encoding**: For each shot, encode at many (resolution, bitrate) combinations. Plot quality (VMAF) vs. bitrate. The convex hull of this plot gives the optimal operating points.
4. **Bitrate Allocation**: Given a target bandwidth budget, allocate more bits to complex shots and fewer to simple shots. Each shot is encoded at its optimal (resolution, bitrate) point.
5. **Assembly**: Stitch shot-level encodes back together with seamless transitions.

**Results**: 20% average bandwidth reduction at the same VMAF quality vs. fixed-ladder encoding. For simple content (animation, documentaries), savings reach 50%.

### Rendition Ladder (Per-Shot)
| Resolution | Typical Bitrate Range | Notes |
|---|---|---|
| 240p | 100-200 Kbps | Cellular fallback |
| 360p | 200-500 Kbps | Low bandwidth |
| 480p | 500-1,000 Kbps | SD |
| 720p | 1,000-3,000 Kbps | HD |
| 1080p | 3,000-6,000 Kbps | Full HD |
| 4K HDR | 8,000-16,000 Kbps | Premium tier |

### Codec Strategy
- **H.264**: Universal compatibility (all devices).
- **VP9/AV1**: 30-50% better compression; used on supported devices (Chrome, Android, Smart TVs).
- **HEVC (H.265)**: Used for 4K HDR on iOS, Apple TV, Xbox.
- **Device capability negotiation**: The Playback Service checks device DRM and codec capabilities, returning only supported renditions in the manifest.

### Encoding Farm
- Encoding a single title at all renditions with per-shot optimization requires ~1,000 GPU-hours.
- Netflix's encoding farm runs on AWS with thousands of GPU instances.
- Encoding is a one-time cost amortized over millions of streams, so high investment per title is justified.

---

## 8) Deep Dive #2: Open Connect CDN Architecture (5-8 min)

Netflix built its own CDN (Open Connect) rather than relying on third-party CDNs. This is the system delivering 150 Tbps at peak.

### Architecture
- **Open Connect Appliances (OCAs)**: Custom hardware (high-capacity SSD/HDD servers) deployed inside ~1,000 ISP networks worldwide.
- **Fill pipeline**: During off-peak hours (2-6 AM local time), OCAs are "filled" with content predicted to be popular. New releases, trending titles, and personalized predictions drive fill decisions.
- **Serving**: During playback, the client streams directly from the OCA inside their ISP. Traffic never crosses the ISP's peering links, reducing costs for both Netflix and the ISP.

### Steering Service
When a playback session starts:
1. Identify the client's ISP and geographic location.
2. Query the **Steering Service** for the best OCA:
   - Does the nearest OCA have the content? (Cache hit)
   - What is the OCA's current load?
   - What is the network path quality (measured via periodic probes)?
3. Return a ranked list of OCA URLs to the client.
4. The client tries the first OCA; if it fails, it falls back to the next.

### Content Placement Algorithm
Not all 15,000 titles fit on every OCA. The placement algorithm:
1. **Popularity prediction**: Predict next-day viewership per title per ISP region.
2. **Capacity constraints**: Each OCA has finite storage (e.g., 100 TB).
3. **Optimization**: Maximize expected cache hit rate given capacity. Popular titles are placed on every OCA; niche titles only on regional OCAs.
4. **Fill scheduling**: Transfer content during off-peak hours to avoid competing with live traffic.

### Fallback Path
```
Client -> ISP OCA (hit 95%+) -> IXP OCA (regional, hit 4%) -> S3 Origin (hit <1%)
```

### Why Build a Custom CDN?
- **Cost**: At Netflix's scale, third-party CDN costs would exceed $1 billion/year. Open Connect is dramatically cheaper.
- **Quality**: Netflix controls the hardware, software, and placement, enabling tighter quality optimization.
- **ISP incentive**: ISPs save on peering costs by hosting OCAs; Netflix gets better performance. A win-win.

---

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Chosen | Alternative | Rationale |
|---|---|---|---|
| CDN strategy | Custom (Open Connect) | Third-party (Akamai, CloudFront) | Cost savings >10x at Netflix scale; tighter quality control |
| Encoding | Per-shot optimization | Fixed bitrate ladder | 20-50% bandwidth savings; justified for known catalog |
| Recommendation approach | Hybrid (collaborative + content-based + contextual) | Collaborative filtering only | Hybrid handles cold-start (new titles, new users) better |
| Streaming protocol | DASH (primary) + HLS (Apple) | HLS-only | DASH is more flexible for DRM and multi-codec; HLS required for Apple ecosystem |
| State storage | Cassandra (watch history) | DynamoDB | Cassandra proven at Netflix scale; operational expertise in-house |

---

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (300M concurrent streams)
- **OCA fleet expansion**: Deploy higher-density OCAs (1 PB storage each) and expand to more ISPs.
- **AV1 adoption**: Encode entire catalog in AV1, reducing bandwidth by 30%+ (equivalent to 30% more capacity without hardware changes).
- **Edge steering improvements**: ML-based OCA selection considering real-time congestion, not just proximity.
- **Recommendation serving**: Precompute and cache personalized feeds; serve from edge caches.

### 100x (hypothetical global dominance)
- **Satellite/mesh delivery**: For regions with poor internet, use peer-to-peer mesh or satellite-assisted delivery.
- **Neural compression**: Per-title neural codecs that learn the content's statistics for extreme compression.
- **On-device caching**: Predictive pre-download of likely-to-be-watched content on user devices during charging/Wi-Fi.
- **Federated recommendation**: Train recommendation models on-device for privacy and reduced server load.
- **Multi-region origins**: Content stored in regional S3 buckets to reduce cross-continent fill traffic.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **OCA failure**: Steering Service detects unhealthy OCAs (health checks every 10s) and redirects clients to the next available OCA. Multiple OCAs per ISP provide redundancy.
- **S3 origin failure**: S3 provides 11 9s durability. Cross-region replication for disaster recovery. OCAs serve as a massive distributed cache, insulating users from origin issues.
- **Microservice failure (Playback, Metadata, Reco)**: Netflix uses Hystrix/Resilience4j for circuit breaking. Degraded mode: show cached/precomputed recommendations if the recommendation service is down; playback continues since manifest URLs are already issued.
- **Regional outage**: DNS-based failover to a secondary AWS region. OCAs continue serving cached content independently of the control plane.
- **DRM license server failure**: Licenses are cached on the client for the session duration. Short outages are invisible to users. Longer outages degrade new session starts only.
- **Chaos engineering**: Netflix pioneered Chaos Monkey (random instance termination), Chaos Kong (region evacuation), and Chaos Gorilla (AZ failure). Continuous resilience testing in production.

---

## 12) Observability and Operations (2-4 min)

- **Stream Starts Per Second (SPS)**: Real-time dashboard; primary health metric. Drop in SPS = immediate investigation.
- **Play delay**: Time from play press to first frame. Target < 2s.
- **Rebuffer rate**: Fraction of playback time spent buffering. Tracked per ISP, per device, per title.
- **Video quality score**: Real-time VMAF estimation based on bitrate and content type.
- **OCA health**: Disk utilization, network throughput, error rates per appliance.
- **Content fill completion**: Percentage of predicted content successfully pre-positioned on OCAs before prime time.
- **A/B test metrics**: Every change (algorithm, UI, encoding) is A/B tested. Primary metrics: retention, engagement, quality of experience.

---

## 13) Security (1-3 min)

- **DRM**: Widevine (Android, Chrome), FairPlay (iOS, Safari, Apple TV), PlayReady (Windows, Xbox). Content encrypted with AES-128. License keys delivered via DRM license server.
- **Manifest signing**: Signed, time-limited URLs prevent unauthorized access and hotlinking.
- **Account security**: Password hashing (bcrypt), multi-factor authentication, device management.
- **Screen limits**: Enforce concurrent stream limits per subscription tier (1/2/4 screens).
- **Geo-restriction**: IP-based geo-lookup enforces regional content licensing. VPN detection to prevent circumvention.
- **Content watermarking**: Forensic watermarks embedded in video to trace piracy to specific accounts.

---

## 14) Team and Operational Considerations (1-2 min)

- **Encoding team**: Owns encoding pipeline, codec R&D (AV1 optimization), per-shot encoding algorithms.
- **Open Connect team**: Manages OCA hardware, fill algorithms, ISP partnerships.
- **Playback/Client team**: Owns player SDKs (web, iOS, Android, TV), ABR algorithm, DRM integration.
- **Recommendations team**: ML model development, A/B testing, personalization pipeline.
- **Reliability team**: Chaos engineering, incident management, capacity planning.
- **Rollout**: Canary releases to 1% of users per device type. Feature flags for gradual rollout.

---

## 15) Common Follow-up Questions

**Q: How does Netflix handle the "cold start" problem for new titles?**
A: (1) Content-based features: genre, cast, director, trailer engagement predict initial interest. (2) Editorial placement in prominent rows for first 48 hours. (3) Rapid feedback loop: after a few thousand views, collaborative filtering signals kick in and refine placement.

**Q: How does "Continue Watching" work across devices?**
A: The client sends periodic heartbeat events with the current playback position. These are written to Cassandra (keyed by profile_id, title_id). When the user opens any device, the client fetches the latest position and resumes. Conflict resolution: latest timestamp wins.

**Q: How does Netflix handle live events (e.g., live comedy specials)?**
A: Live content uses a separate pipeline: RTMP/SRT ingest -> live transcoder (low-latency encoding) -> segmented and pushed to OCAs in real-time. Latency target: 10-30 seconds (not real-time like Twitch, but acceptable for scheduled events).

**Q: Why does Netflix use microservices so heavily?**
A: With 200+ microservices, each team owns their service end-to-end (deployment, scaling, on-call). This allows independent scaling (recommendation service scales differently than playback) and rapid deployment (1000+ deployments/day across services).

**Q: How are thumbnails personalized?**
A: Netflix generates multiple artwork variants per title. An ML model selects the variant most likely to drive engagement for each user based on their viewing history (e.g., a romance viewer sees the romantic leads; an action viewer sees an explosion scene).

---

## 16) Closing Summary (30-60s)

> "We designed a Netflix-scale streaming platform serving 250 million subscribers with 30 million concurrent streams. The encoding pipeline uses per-shot optimization to reduce bandwidth 20-50% at target VMAF quality. Content is pre-positioned on ISP-embedded Open Connect appliances during off-peak hours, achieving 95%+ cache hit rates and handling 150 Tbps peak egress. The home page is driven by a hybrid recommendation engine that personalizes rows, thumbnails, and rankings. Playback orchestration handles DRM license issuance, CDN server selection via the Steering Service, and cross-device resume. The system is battle-tested with chaos engineering and A/B tests every change."

---

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tune When |
|---|---|---|
| `per_shot_complexity_threshold` | adaptive | Lower for more aggressive optimization |
| `oca_fill_window` | 2-6 AM local | Adjust for ISP traffic patterns |
| `oca_cache_capacity` | 100 TB | Higher-density hardware for top ISPs |
| `abr_buffer_target` | 40s | Lower for faster start; higher for fewer rebuffers |
| `drm_license_duration` | 24h | Shorter for tighter security; longer for offline resilience |
| `concurrent_stream_limit` | 1/2/4 | Per subscription tier |
| `recommendation_row_count` | 40 | More rows for engagement; fewer for simplicity |
| `fill_popularity_threshold` | top 20% | More titles pre-positioned vs. storage cost |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|---|---|
| **Open Connect** | Netflix's custom CDN with appliances deployed inside ISP networks |
| **OCA** | Open Connect Appliance; a server inside an ISP serving video segments |
| **Per-Shot Encoding** | Optimizing encoding parameters per scene/shot rather than per title |
| **VMAF** | Video Multimethod Assessment Fusion; Netflix's perceptual quality metric |
| **Mezzanine** | Studio-quality master file used as the source for encoding |
| **Convex Hull Encoding** | Finding optimal quality-vs-bitrate operating points per shot |
| **Steering Service** | Selects the best OCA for each playback session |
| **ABR** | Adaptive Bitrate; client dynamically selects quality based on bandwidth |
| **DRM** | Digital Rights Management; encrypts content to prevent piracy |
| **Forensic Watermark** | Invisible per-user marker embedded in video for piracy tracing |

## Appendix C: References

- Netflix Open Connect Overview (Netflix Tech Blog)
- Per-Title Encode Optimization (Netflix Tech Blog, 2015)
- Dynamic Optimizer (Netflix, per-shot encoding evolution)
- Netflix Recommendations: Beyond the 5 Stars (2012)
- Zuul 2: Netflix's API Gateway
- Chaos Engineering: Netflix's approach to resilience testing
- AV1 Adoption at Netflix
