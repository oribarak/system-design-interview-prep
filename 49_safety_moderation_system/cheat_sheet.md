# Safety / Moderation Filtering System — Cheat Sheet

## Key Numbers
- 2B text items/day, 500M images/day, 50M videos/day
- 23K avg text QPS, 70K peak; 6K image QPS
- < 100ms p99 inline text moderation; < 5s image classification
- ~350 GPUs for text, ~300 GPUs for image inference
- 0.5% flagged for human review = 12.5M items/day
- < 1% false-positive rate for auto-block decisions

## Core Components
- **Hash Matcher**: PhotoDNA/pHash lookup against known-bad content (< 1ms)
- **Rule Engine**: deterministic keyword/regex/rate-limit checks (< 5ms)
- **ML Classifiers**: DistilBERT (text), EfficientNet (image), keyframe-based (video) on GPU fleet
- **Decision Engine**: applies per-category policy thresholds to ML scores
- **Human Review Queue**: Redis Sorted Sets, priority by severity, SLA-tracked
- **Policy Console**: DSL-based rules editable by T&S teams without code deploys

## Architecture in One Sentence
Content flows through a cascading pipeline of hash matching, rule engine, ML classification, and human review, with a policy DSL controlling thresholds and actions at each stage.

## Top 3 Trade-offs
1. **Inline vs async**: inline for text (pre-publish safety) vs async for media (latency budget)
2. **Fail-open vs fail-closed**: per-category decision -- CSAM always fail-closed, spam fail-open
3. **Custom ML vs third-party APIs**: custom for policy-specific tuning, third-party as fallback

## Scaling Story
- 10x: GPU autoscaling, model distillation, decision caching for duplicate content
- 100x: tiered CPU pre-filter (80% clearly safe skips GPU), edge inference, ML-assisted human review

## Closing Statement
"Multi-stage cascade (hash, rules, ML, human) provides defense-in-depth. Policy DSL decouples T&S iteration from engineering. Calibrated thresholds and active learning continuously improve accuracy. Fail-mode matrix balances safety vs availability per violation category."
