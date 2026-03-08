# System Design Interview: Safety / Moderation Filtering System — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a safety and moderation filtering system that screens user-generated content -- text, images, and video -- for policy violations such as hate speech, spam, CSAM, violence, and misinformation. The system must operate at scale with low-latency inline filtering for real-time content, asynchronous deep analysis for uploaded media, a human review pipeline for ambiguous cases, and a policy management layer that lets trust-and-safety teams update rules without code deployments. I'll walk through the ML classification pipeline, the rule engine, appeal workflows, and how we scale this to billions of content items per day."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What content types? Text, images, video, audio? | All four, but text and images are highest volume |
| 2 | Is moderation inline (pre-publish) or async (post-publish)? | Both: lightweight inline for text; async deep scan for media |
| 3 | What policy categories? Hate speech, spam, NSFW, CSAM, violence, misinformation? | All of the above, extensible |
| 4 | What is the acceptable false-positive rate? | < 1% for auto-removal; higher tolerance for flagging to human review |
| 5 | Do we need multi-language support? | Yes, at least top 20 languages |
| 6 | Is there an appeals process? | Yes, users can appeal automated decisions |
| 7 | Are there legal/regulatory requirements (GDPR, DSA, local laws)? | Yes, region-specific compliance |
| 8 | What latency is acceptable for inline moderation? | < 100ms p99 for text; < 5s for image |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Daily content items | 2B text posts, 500M images, 50M videos |
| Inline text moderation QPS | 2B / 86,400 ~ 23K QPS avg, 70K QPS peak |
| Image moderation QPS | 500M / 86,400 ~ 6K QPS avg |
| Avg text payload | 500 bytes |
| Avg image payload | 200 KB (thumbnail for classification) |
| ML inference per text item | ~5ms on GPU |
| ML inference per image | ~50ms on GPU |
| GPU fleet for text (peak) | 70K * 5ms = 350 GPU-seconds/s ~ 350 GPUs |
| GPU fleet for images | 6K * 50ms = 300 GPU-seconds/s ~ 300 GPUs |
| Human review queue | ~0.5% flagged = 12.5M items/day |
| Human reviewers needed | 12.5M / (200 reviews/reviewer/day) = ~62,500 reviewers (outsourced) |
| Decision log storage/day | 2.5B * 1 KB = 2.5 TB/day |
| Policy rules | 500-2,000 active rules |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Inline text moderation**: classify text content and return allow/flag/block decision within 100ms
2. **Async media moderation**: classify images and video for policy violations within 5s (images), 60s (video)
3. **Multi-stage pipeline**: rule engine, ML classifiers, and human review in cascade
4. **Policy management console**: trust-and-safety teams can create/update/disable rules without code changes
5. **Appeals workflow**: users can appeal decisions; reviewers can overturn
6. **Audit trail**: every decision logged with reasoning, model version, and confidence score

### Non-Functional Requirements
- **Latency**: < 100ms p99 for inline text; < 5s for image classification
- **Availability**: 99.99% -- downtime means unmoderated content goes live
- **Accuracy**: < 1% false-positive rate for auto-block; < 5% false-negative rate for high-severity categories
- **Consistency**: same content should receive same decision across retries
- **Scalability**: handle 10x traffic spikes during viral events
- **Compliance**: GDPR right-to-erasure, DSA transparency reporting

## 4) API Design (2-4 min)

### Inline Moderation (synchronous)
```
POST /v1/moderate/text
{
  "content_id": "uuid",
  "text": "string",
  "content_type": "post|comment|message",
  "user_id": "string",
  "locale": "en-US",
  "context": { "subreddit": "...", "channel_id": "..." }
}
Response: {
  "decision": "ALLOW | FLAG | BLOCK",
  "violated_policies": ["hate_speech"],
  "confidence": 0.92,
  "request_id": "uuid"
}
```

### Async Media Moderation
```
POST /v1/moderate/media
{
  "content_id": "uuid",
  "media_url": "s3://...",
  "media_type": "image|video|audio",
  "user_id": "string",
  "callback_url": "https://..."
}
Response: { "job_id": "uuid", "status": "QUEUED" }
```

### Appeal
```
POST /v1/appeals
{
  "content_id": "uuid",
  "user_id": "string",
  "reason": "string"
}
```

### Policy Management (internal)
```
POST /v1/policies
PUT /v1/policies/{policy_id}
GET /v1/policies?category=hate_speech&status=active
```

## 5) Data Model (3-5 min)

### Core Entities

**ModerationDecision** (primary write path -- append-only log)
| Column | Type | Notes |
|--------|------|-------|
| decision_id | UUID | PK |
| content_id | UUID | FK, indexed |
| user_id | string | indexed |
| decision | enum | ALLOW, FLAG, BLOCK |
| stage | enum | RULE_ENGINE, ML, HUMAN |
| violated_policies | string[] | policy IDs |
| confidence | float | ML confidence score |
| model_version | string | model lineage |
| created_at | timestamp | partition key |

**Policy**
| Column | Type | Notes |
|--------|------|-------|
| policy_id | UUID | PK |
| category | string | hate_speech, spam, etc. |
| rule_type | enum | REGEX, KEYWORD, ML_THRESHOLD, HASH |
| rule_config | JSON | thresholds, patterns |
| severity | enum | LOW, MEDIUM, HIGH, CRITICAL |
| status | enum | ACTIVE, DISABLED, TESTING |
| updated_at | timestamp | |

**Appeal**
| Column | Type | Notes |
|--------|------|-------|
| appeal_id | UUID | PK |
| content_id | UUID | FK |
| user_id | string | |
| status | enum | PENDING, APPROVED, DENIED |
| reviewer_id | string | |
| outcome_reason | string | |

### Storage Choices
- **Decision log**: Cassandra / ScyllaDB -- high write throughput, time-partitioned
- **Policies**: PostgreSQL -- relational, low volume, needs ACID for updates
- **Human review queue**: Redis Sorted Sets -- priority queue by severity
- **Hash databases (CSAM, known-bad)**: RocksDB -- fast hash lookups, billions of entries
- **ML feature store**: Redis -- cached user-reputation scores

## 6) High-Level Architecture (5-8 min)

Content enters the system through API Gateway, passes through a multi-stage pipeline (rule engine, ML classifiers, optional human review), and a decision is returned or pushed via callback. A policy management console lets T&S teams control the pipeline without code changes.

### Pipeline Stages (cascading):
1. **Hash Matching** (< 1ms): check perceptual hashes against known-bad database (PhotoDNA for CSAM)
2. **Rule Engine** (< 5ms): regex, keyword lists, rate-limit checks
3. **ML Classification** (< 50ms): text/image/video classifiers produce per-category scores
4. **Decision Engine** (< 2ms): combine scores with policy thresholds to produce final decision
5. **Human Review** (async): items in the gray zone are queued for human reviewers

### Components:
- **API Gateway**: rate limiting, auth, route to sync/async path
- **Rule Engine Service**: evaluates deterministic rules (keyword blocklists, regex patterns)
- **ML Inference Service**: hosts text/image/video models on GPU fleet
- **Decision Engine**: applies policy thresholds to ML scores, produces final verdict
- **Human Review Service**: manages reviewer queues, assignment, SLA tracking
- **Policy Management Service**: CRUD for policies, A/B testing of new rules
- **Hash DB**: stores perceptual hashes for known-bad content
- **Audit Log**: immutable append-only log of every decision

## 7) Deep Dive #1: ML Classification Pipeline (8-12 min)

### Model Architecture
- **Text classifier**: fine-tuned transformer (distilled BERT, ~66M params) with multi-label heads for each policy category. Quantized to INT8 for inference speed.
- **Image classifier**: EfficientNet-B4 backbone with multi-label heads. Input: 380x380 thumbnail.
- **Video classifier**: sample keyframes (1 per second), run image classifier, aggregate with temporal model.
- **Audio classifier**: Whisper-based transcription followed by text classifier, plus audio-specific models for threats.

### Multi-Label Output
Each model produces a vector of scores: `[hate: 0.87, spam: 0.02, nsfw: 0.91, violence: 0.12, ...]`. The Decision Engine compares each score against the policy-defined threshold for that category.

### Model Serving
- **Triton Inference Server** or **TorchServe** on GPU instances
- **Dynamic batching**: accumulate requests for 5ms, then batch-infer (increases throughput 4-8x)
- **Model versioning**: canary deployments -- 5% traffic to new model, compare metrics before full rollout
- **Fallback**: if GPU fleet is overloaded, degrade to lightweight CPU model (lower accuracy, but keeps latency SLA)

### Confidence Calibration
Raw model scores are not well-calibrated probabilities. Apply Platt scaling (logistic regression on held-out set) to calibrate scores so that a threshold of 0.90 actually means 90% precision.

### Active Learning Loop
1. Human review decisions feed back as training labels
2. Weekly model retraining with fresh labeled data
3. Shadow-mode evaluation: new model runs alongside production model, compare decisions
4. Promote to production only if precision/recall improve on holdout set

### Adversarial Robustness
- **Text**: adversarial typos (h@te, ha.te), Unicode homoglyphs, leetspeak. Mitigate with character normalization and adversarial training data.
- **Images**: steganography, text overlaid on images (need OCR + text classifier). Mitigate with multi-modal models that jointly analyze visual and textual signals.
- **Rate-aware detection**: same user posting borderline content 100x should raise severity.

## 8) Deep Dive #2: Policy Engine and Human Review (5-8 min)

### Policy Engine Architecture
Policies are expressed as a DSL (domain-specific language) that compiles to evaluation rules:

```
policy "hate_speech_auto_block" {
  condition: ml_score("hate_speech") > 0.95 AND user_reputation < 0.3
  action: BLOCK
  severity: CRITICAL
  notify: ["trust_safety_oncall"]
}

policy "spam_flag_for_review" {
  condition: ml_score("spam") > 0.7 AND ml_score("spam") < 0.95
  action: FLAG
  priority: MEDIUM
}
```

Policies are versioned, A/B testable, and can be toggled per-region. The policy engine evaluates all active policies and returns the highest-severity action.

### Human Review Pipeline
1. **Queue Management**: items enter priority queues (Redis Sorted Sets) keyed by severity and SLA deadline
2. **Assignment**: round-robin with specialization -- reviewers are certified for specific categories (e.g., CSAM requires law-enforcement-trained reviewers)
3. **Review Interface**: shows content, ML scores, similar past decisions, policy context
4. **Decision**: reviewer selects APPROVE / REMOVE / ESCALATE with a reason code
5. **Quality Assurance**: 10% of decisions are audited by senior reviewers; inter-rater reliability tracked
6. **SLA Tracking**: CRITICAL items must be reviewed within 1 hour; HIGH within 4 hours; MEDIUM within 24 hours
7. **Reviewer Wellness**: auto-rotate reviewers off graphic content after 4 hours; blur graphic content by default

### Appeal Flow
1. User submits appeal with reason
2. Appeal goes to a different reviewer than the original (avoid confirmation bias)
3. If second reviewer disagrees with original, a third reviewer breaks the tie
4. Appeal decision is final and logged

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| Inline + async hybrid | Fully async (post-publish) | Inline catches worst content before publish; async for expensive media analysis |
| Custom ML models | Third-party APIs (Perspective, AWS Rekognition) | Custom models allow policy-specific tuning and avoid vendor lock-in; can use third-party as fallback |
| Cascading pipeline (rules then ML) | ML-only | Rule engine catches known-bad instantly at near-zero cost; ML handles novel content |
| Policy DSL | Hardcoded thresholds in code | DSL lets T&S teams iterate without engineering bottleneck |
| Calibrated thresholds per category | Single global threshold | Different categories have different cost profiles (CSAM false-negative is catastrophic; spam false-negative is minor) |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (25B items/day)
- **GPU autoscaling**: scale inference fleet based on queue depth; use spot/preemptible instances for async path
- **Model distillation**: distill large models to smaller ones for inline path (3x speedup with < 2% accuracy loss)
- **Regional deployment**: deploy moderation service in each region to reduce cross-region latency
- **Caching**: cache decisions for identical content (hash-based dedup) -- ~15% of content is reposts/copies

### 100x Scale (250B items/day)
- **Tiered classification**: cheap CPU model filters 80% as clearly-safe; only 20% goes to GPU models
- **Edge inference**: deploy lightweight models at CDN edge for pre-filtering
- **Federated model serving**: shard models by content type across specialized GPU clusters
- **Async everything**: move to fully async with optimistic publishing and fast takedown (< 500ms)
- **Invest in human review automation**: ML-assisted review where model pre-fills decision, reviewer confirms (5x throughput)

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| ML service down | Fail-open with enhanced rule engine + alert; or fail-closed for CSAM |
| Policy service down | Cache policies locally with 5-min TTL; stale policies are better than none |
| Human review backlog | Auto-escalate by severity; temporary threshold adjustment to auto-decide more |
| Hash DB unavailable | Queue content for retry; fail-closed for media uploads |
| False positive spike (bad model) | Automatic rollback if block rate exceeds 2x baseline; circuit breaker per model version |
| Regional outage | Route to nearest healthy region; accept higher latency over no moderation |

### Fail-Open vs Fail-Closed Decision Matrix
- **CSAM, terrorism**: always fail-closed (block on doubt)
- **Spam, mild policy violations**: fail-open (allow, review later)
- **Hate speech, violence**: fail-open for text with retroactive scan; fail-closed for images

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **Latency**: p50/p95/p99 per pipeline stage
- **Throughput**: items/sec per content type
- **Decision distribution**: % allow/flag/block per category (anomaly detection on shifts)
- **False positive rate**: tracked via appeal overturn rate
- **False negative rate**: tracked via user reports on allowed content
- **Human review SLA adherence**: % of items reviewed within SLA
- **Model confidence distribution**: histogram of confidence scores (shift detection)

### Dashboards
- Real-time pipeline health (latency, error rate, queue depth)
- Policy effectiveness (precision/recall per policy)
- Reviewer performance (throughput, agreement rate, wellness metrics)
- Regional compliance (decision rates by jurisdiction)

### Alerting
- Block rate exceeds 2x baseline (possible false positive surge)
- Queue depth exceeds 1M items (human review falling behind)
- Model confidence mean drops by > 0.1 (distribution shift)
- Latency p99 exceeds 200ms for inline path

## 13) Security (1-3 min)

- **Content isolation**: reviewers see content in sandboxed viewer to prevent malware execution
- **Access control**: RBAC for policy management; only T&S leads can modify CRITICAL-severity policies
- **Data retention**: moderation decisions retained for regulatory compliance (7 years); actual content referenced by ID only
- **PII handling**: user IDs in decision logs are pseudonymized; real identity only accessible via secure lookup
- **CSAM handling**: content is never stored in cleartext; hashed and reported to NCMEC per legal requirements
- **Audit logging**: all policy changes, reviewer actions, and system overrides are logged immutably
- **Rate limiting**: moderation API rate-limited per tenant to prevent abuse

## 14) Team and Operational Considerations (1-2 min)

- **ML Team (5-8 engineers)**: model training, evaluation, deployment, adversarial robustness
- **Platform Team (4-6 engineers)**: pipeline infrastructure, scaling, latency optimization
- **Policy/T&S Team (3-5 engineers)**: policy DSL, rule engine, policy console UI
- **Human Review Ops (2-3 managers + outsourced reviewers)**: queue management, QA, reviewer wellness
- **On-call rotation**: separate for ML service, pipeline, and human review
- **Runbooks**: automated rollback for bad model deployments; manual escalation for regulatory incidents

## 15) Common Follow-up Questions

**Q: How do you handle new policy categories that models haven't been trained on?**
A: Start with keyword/rule-based detection, collect labeled data from human reviewers, train a model after accumulating ~10K labeled examples, then promote to ML pipeline.

**Q: How do you prevent gaming/adversarial attacks?**
A: Multi-signal approach: combine text analysis with user reputation score, posting velocity, account age. Adversarial training data augmentation. Regular red-team exercises.

**Q: How do you handle context-dependent content (e.g., news reporting on violence)?**
A: Content metadata (source, user type, channel) feeds into the decision engine. Verified news accounts have higher thresholds. Context features are part of the ML input.

**Q: How do you balance free speech vs safety?**
A: Configurable per-region policies aligned with local law (e.g., EU DSA requirements). Tiered actions: warn before block. Transparent community guidelines.

**Q: What about real-time content like live streams?**
A: Sample frames/audio at 1 FPS, run through image/audio classifiers with 2-3 second lag. Auto-cut stream if CRITICAL violation detected. Higher tolerance thresholds to account for lower confidence on single frames.

## 16) Closing Summary (30-60s)

"I've designed a multi-stage safety and moderation system that cascades through hash matching, rule engine, ML classification, and human review. The system handles 2B+ daily content items with < 100ms inline text moderation. Key design choices include a policy DSL for non-engineering iteration, calibrated ML models with active learning, and a fail-open/fail-closed matrix based on violation severity. The architecture scales to 100x through tiered classification, edge inference, and ML-assisted human review. Observability focuses on decision distribution anomaly detection and false-positive tracking via appeal rates."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| ml_confidence_threshold (per category) | 0.90 | 0.5 - 0.99 | Higher = fewer false positives, more false negatives |
| dynamic_batch_window_ms | 5 | 1 - 20 | Higher = better GPU utilization, higher latency |
| human_review_sla_critical_hours | 1 | 0.5 - 4 | Lower = faster response, more reviewer pressure |
| fail_mode (per category) | OPEN | OPEN / CLOSED | Whether to allow or block content when ML is unavailable |
| cache_ttl_identical_content_sec | 3600 | 300 - 86400 | Higher = more cache hits, staler decisions |
| reviewer_rotation_hours | 4 | 2 - 8 | Hours before rotating reviewer off graphic content |
| canary_traffic_pct | 5 | 1 - 50 | Percentage of traffic to new model version |
| tiered_cpu_filter_threshold | 0.1 | 0.05 - 0.3 | Below this score, skip GPU inference (clearly safe) |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| PhotoDNA | Microsoft perceptual hashing technology for detecting known CSAM images |
| Platt Scaling | Post-hoc calibration method that maps model outputs to true probabilities |
| Dynamic Batching | Accumulating inference requests into a batch for GPU efficiency |
| Fail-Open | Allow content through when moderation system is unavailable |
| Fail-Closed | Block content when moderation system is unavailable |
| DSA | EU Digital Services Act -- requires content moderation transparency |
| NCMEC | National Center for Missing & Exploited Children -- CSAM reporting body |
| Active Learning | ML training loop where human review decisions become new training data |
| Adversarial Robustness | Model's resistance to intentional manipulation of input to evade detection |
| Policy DSL | Domain-specific language for expressing moderation rules declaratively |

## Appendix C: References

- Meta Transparency Center: content moderation at scale
- Google Jigsaw Perspective API: toxic comment classification
- Microsoft PhotoDNA: perceptual hash matching
- EU Digital Services Act: regulatory framework for content moderation
- "The Internet's Hidden Rules" (Stanford Internet Observatory)
- Triton Inference Server documentation for model serving
- "Measuring and Mitigating Bias in Content Moderation" (ACM FAccT)
