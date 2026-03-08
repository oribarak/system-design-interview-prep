# Experimentation & Evaluation Platform -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a platform that lets teams run A/B tests on LLM prompt variants, model versions, and configurations while measuring quality through automated scoring (including LLM-as-judge), human evaluation, and statistical analysis. The hard part is ensuring statistically valid results, keeping scoring costs manageable, and providing real-time experiment monitoring with guardrails.

## 2. Requirements
**Functional:** Define experiments with variants, traffic splits, and metrics. Deterministically route users to variants with sticky bucketing. Run offline evaluation against golden datasets as a quality gate. Score responses via LLM-as-judge and human raters. Compute statistical significance and declare winners.

**Non-functional:** Less than 5ms routing overhead. Metrics dashboards updated within 5 minutes. Support 200+ concurrent experiments. Same user always sees same variant. Full audit lineage from experiment config to outcome.

## 3. Core Concept: Multi-Layered Evaluation
Quality is measured through three layers of increasing cost and accuracy. Automated metrics (latency, format checks, embedding similarity) are instant and free. LLM-as-judge scores (relevance, coherence, safety) run in seconds using a fine-tuned small model at 10x less cost than a frontier model, with 5% sampled to an expensive model for calibration. Human evaluation on 3-5% of traffic validates everything else. This layered approach keeps costs manageable while ensuring accuracy.

## 4. High-Level Architecture
```
LLM App --> [Assignment Service] --> variant config
                  |
           [Outcome Collector] --> Kafka
                                     |
              [Scoring Pipeline] -- automated + LLM judge + human sample
                                     |
              [Analytics Engine] --> ClickHouse --> Dashboard
                                     |
              [Guardrail Monitor] -- auto-pause on quality regression
```
- **Assignment Service**: Hash(user_id + experiment_id) mod 100 for deterministic sticky bucketing.
- **Scoring Pipeline**: Flink streaming job that runs automated metrics inline and dispatches to LLM-judge workers.
- **Analytics Engine**: Aggregates scores per variant, runs statistical tests (t-test, sequential analysis).
- **Guardrail Monitor**: Auto-pauses experiments if error rate or latency exceeds thresholds within 1 minute.

## 5. How an Experiment Runs
1. Team defines experiment: variants, primary metric, guardrails, target sample size.
2. Offline evaluation runs against golden datasets as a quality gate before live traffic.
3. Experiment launches; Assignment Service routes users deterministically to variants.
4. Outcomes (request, response, latency) flow to Kafka and through the scoring pipeline.
5. LLM-as-judge scores each response on quality dimensions; human raters evaluate a 3-5% sample.
6. Analytics Engine computes confidence intervals using sequential analysis, allowing early stopping.
7. When statistical significance is reached, experiment concludes and winner is declared.

## 6. What Happens When Things Fail?
- **Assignment service down**: Fall back to control variant (safe default). Deterministic hashing is stateless so any replica gives the same answer.
- **Judge model unavailable**: Queue outcomes for retry. Dashboards show stale data with a warning.
- **Bad experiment harms quality**: Guardrail monitor auto-pauses treatment within 1 minute. Error rate, latency, and safety score are all monitored.

## 7. Scaling
- **10x**: Partition Kafka by experiment_id for parallel scoring. Autoscale judge model GPU fleet based on queue depth. Shard ClickHouse by experiment_id.
- **100x**: Sample 10% for judge evaluation and 1% for human eval instead of scoring everything. Push assignment logic to edge proxies. Tiered storage with hot data in ClickHouse, warm in Parquet on S3.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Fine-tuned small model as judge | 10x cheaper than frontier model with less than 5% accuracy loss; frontier used for calibration |
| Sequential analysis over fixed-horizon | Allows early stopping, reducing experiment duration by 30-50%, but more complex to implement |
| Sticky user-level bucketing | Consistent user experience but less statistical power than request-level randomization |

## 9. Closing (30s)
> This platform has three pillars: deterministic traffic routing with sticky bucketing, a multi-layered evaluation pipeline (automated metrics, LLM-as-judge, human raters), and rigorous statistical analysis with sequential testing and automatic guardrails. Offline evaluation gates experiments before live traffic. The key cost insight is using a fine-tuned small model as the primary judge with frontier model calibration on a 5% sample.
