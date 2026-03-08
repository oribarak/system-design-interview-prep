# Experimentation & Evaluation Platform for LLMs — Cheat Sheet

## Key Numbers
- 10M requests/day across experiments, 115 QPS avg, 350 peak
- 100 concurrent experiments, 50-200 target
- LLM-as-judge: fine-tuned 8B model at $0.001/call; 5% calibration on frontier model
- Human eval on 3-5% of traffic; 2+ raters per item
- Statistical power: 2K-10K samples per variant for 2% MDE
- Metrics dashboard refresh: 5-minute windows
- Guardrail check interval: 1 minute

## Core Components
- **Assignment Service**: deterministic hash-based routing, sticky bucketing via Redis
- **Outcome Collector**: ingests metrics + payloads to Kafka
- **LLM-as-Judge**: fine-tuned small model scores relevance/coherence/safety; frontier model for calibration
- **Human Eval Service**: structured rubric-based evaluation on sampled traffic
- **Analytics Engine**: sequential statistical analysis (O'Brien-Fleming), confidence intervals
- **Guardrail Monitor**: auto-pauses experiments exceeding error/latency/safety thresholds
- **Offline Eval Runner**: batch evaluation on golden datasets as quality gate

## Architecture in One Sentence
Deterministic traffic routing assigns users to experiment variants, outcomes stream through Kafka to a multi-layer scoring pipeline (automated + LLM judge + human), and sequential statistical analysis declares winners while guardrails auto-pause degraded variants.

## Top 3 Trade-offs
1. **Fine-tuned small judge vs frontier model**: 10x cheaper with 5% calibration sample ensuring accuracy
2. **Sequential vs fixed-horizon testing**: allows early stopping (30-50% faster) with controlled error rate
3. **User-level vs request-level bucketing**: user-level ensures consistent experience at cost of statistical power

## Scaling Story
- 10x: partition Kafka by experiment_id, autoscale judge GPU fleet, shard ClickHouse
- 100x: sample 10% for judge scoring, edge-based assignment, tiered storage (hot/warm/cold)

## Closing Statement
"Three pillars: deterministic routing with sticky bucketing, multi-layered evaluation (automated + LLM judge + human), and rigorous sequential statistics with guardrails. Offline eval gates prevent regressions before live traffic. Fine-tuned small-model judges provide 10x cost savings over frontier models."
