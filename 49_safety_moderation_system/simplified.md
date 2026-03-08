# Safety / Moderation System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a system that screens billions of daily user-generated content items (text, images, video) for policy violations like hate speech, spam, and CSAM. The hard part is doing this at low latency for inline content while maintaining high accuracy, handling ambiguous cases through human review, and letting trust-and-safety teams update policies without code deployments.

## 2. Requirements
**Functional:** Inline text moderation returning allow/flag/block within 100ms. Async media moderation for images and video. A cascading pipeline of rule engine, ML classifiers, and human review. A policy management console for non-engineering updates. An appeals workflow for users to contest decisions.

**Non-functional:** Less than 100ms p99 for inline text, less than 5s for images. 99.99% availability. Less than 1% false-positive rate for auto-block. Handle 10x traffic spikes. GDPR and DSA compliance.

## 3. Core Concept: The Cascading Pipeline
Content flows through stages of increasing cost and accuracy. First, hash matching checks against known-bad content in under 1ms. Then, a rule engine evaluates keyword blocklists and regex patterns in under 5ms. Next, ML classifiers produce per-category confidence scores in under 50ms. Finally, a decision engine applies policy thresholds. Items in the gray zone go to human review. Each stage short-circuits -- if hash matching catches CSAM, we skip everything else.

## 4. High-Level Architecture
```
Content --> [API Gateway] --> [Hash DB Check]
                                  |
                          [Rule Engine] --> [ML Classifiers (GPU)]
                                                   |
                                           [Decision Engine] --> Allow / Block
                                                   |
                                           [Human Review Queue] (gray zone)
```
- **Hash DB**: Perceptual hashes for instant known-bad content detection (PhotoDNA for CSAM).
- **Rule Engine**: Deterministic regex/keyword checks, near-zero cost.
- **ML Classifiers**: Transformer-based text models, EfficientNet for images, producing multi-label scores.
- **Decision Engine**: Combines ML scores with policy thresholds configured by trust-and-safety teams.
- **Human Review Queue**: Priority queue in Redis sorted by severity and SLA deadline.

## 5. How Content Moderation Works
1. Content arrives at API Gateway (text inline, media async via callback).
2. Hash matching checks perceptual hashes against known-bad database.
3. Rule engine evaluates keyword blocklists and regex patterns.
4. ML classifiers produce per-category scores (hate: 0.87, spam: 0.02, etc.).
5. Decision engine compares scores against policy-defined thresholds.
6. High-confidence violations are auto-blocked; low-confidence items are allowed; gray zone goes to human review.
7. Every decision is logged with model version, confidence, and reasoning for audit.

## 6. What Happens When Things Fail?
- **ML service down**: Fail-open for spam (allow, review later) but fail-closed for CSAM and terrorism (block on doubt). Enhanced rule engine covers the gap.
- **Bad model deployment (false positive spike)**: Automatic rollback if block rate exceeds 2x baseline. Circuit breaker per model version.
- **Human review backlog**: Auto-escalate by severity. Temporarily adjust thresholds to auto-decide more items.

## 7. Scaling
- **10x**: GPU autoscaling based on queue depth. Model distillation for inline path. Cache decisions for identical content (hash-based dedup handles ~15% reposts). Regional deployment.
- **100x**: Tiered classification where a cheap CPU model filters 80% as clearly safe, only 20% goes to GPU. Edge inference for pre-filtering. ML-assisted human review where the model pre-fills decisions and reviewers confirm (5x throughput).

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Inline + async hybrid pipeline | Catches worst content pre-publish; async handles expensive media analysis |
| Cascading rules-then-ML | Rules catch known-bad instantly at near-zero cost; ML handles novel content |
| Policy DSL for T&S teams | Lets non-engineers iterate on rules without code deployments, but adds DSL maintenance cost |

## 9. Closing (30s)
> This is a multi-stage cascading pipeline that moves content through hash matching, rule engine, ML classification, and human review. The key insight is fail-open vs fail-closed by category -- CSAM is always blocked on doubt while spam is allowed and reviewed later. A policy DSL lets trust-and-safety teams update rules without engineering involvement, and active learning from human review decisions continuously improves ML models.
