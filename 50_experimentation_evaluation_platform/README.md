# System Design Interview: Experimentation & Evaluation Platform for LLMs — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design an experimentation and evaluation platform purpose-built for LLM applications. This system lets teams run A/B tests on prompt variants, model versions, and configuration changes while measuring quality metrics like relevance, coherence, safety, and latency. The platform includes offline evaluation with golden datasets, online experimentation with traffic splitting, automated scoring via judge LLMs, human evaluation workflows, and statistical analysis for determining winners. I'll focus on the evaluation pipeline architecture, experiment lifecycle management, and how we ensure statistically valid results."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What is being experimented on? Prompts, models, RAG configs, guardrails? | All of the above -- any variable in the LLM pipeline |
| 2 | What scale? How many experiments run concurrently? | 50-200 concurrent experiments across teams |
| 3 | How is quality measured? Automated metrics, human eval, or both? | Both: automated scorers (including LLM-as-judge) plus human evaluation |
| 4 | What traffic volume goes through the experimented LLM pipelines? | ~10M requests/day across all experiments |
| 5 | Do we need offline (batch) evaluation on test datasets? | Yes, pre-launch quality gates before online experiments |
| 6 | Statistical rigor requirements? | Standard A/B testing with sequential analysis for early stopping |
| 7 | Multi-tenancy? Multiple teams running independent experiments? | Yes, full isolation between teams/products |
| 8 | Integration with existing ML platforms? | Should integrate with model registries and feature stores |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Concurrent experiments | 100 active |
| Online traffic | 10M requests/day ~ 115 QPS avg, 350 QPS peak |
| Avg LLM response size | 500 tokens ~ 2 KB |
| Evaluation data per request | request + response + scores ~ 5 KB |
| Daily evaluation storage | 10M * 5 KB = 50 GB/day |
| Offline eval dataset sizes | 1K - 50K examples per dataset |
| LLM-as-judge calls per request | 2-3 scoring dimensions = 30M judge calls/day |
| Judge cost (GPT-4 class) | 30M * $0.01 = $300K/day -- need cheaper judges |
| Judge cost (fine-tuned small model) | 30M * $0.001 = $30K/day |
| Human eval budget | 5% of traffic = 500K evals/day |
| Statistical sample size per variant | ~2,000-10,000 for detecting 2% quality difference |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Experiment definition**: configure variants (prompt, model, params), traffic split, metrics, guardrails
2. **Traffic routing**: deterministically route users to experiment variants based on hashing
3. **Offline evaluation**: run variants against golden test datasets with automated scoring
4. **Online evaluation**: collect real-time metrics from production traffic
5. **Automated scoring**: LLM-as-judge scores for quality dimensions (relevance, coherence, safety)
6. **Human evaluation**: sample traffic to human raters with structured evaluation rubrics
7. **Statistical analysis**: compute confidence intervals, p-values, determine experiment winners
8. **Experiment lifecycle**: create, launch, ramp, pause, conclude experiments with approval gates

### Non-Functional Requirements
- **Latency overhead**: < 5ms added to request path for experiment routing
- **Consistency**: same user always sees same variant (sticky bucketing)
- **Isolation**: experiments don't interfere with each other; guardrails halt bad variants
- **Freshness**: metrics dashboards update within 5 minutes
- **Scalability**: support 200+ concurrent experiments
- **Auditability**: full lineage from experiment config to outcome decision

## 4) API Design (2-4 min)

### Experiment Management
```
POST /v1/experiments
{
  "name": "prompt_v2_test",
  "hypothesis": "New prompt improves relevance by 5%",
  "variants": [
    { "name": "control", "config": { "prompt_template_id": "v1", "model": "gpt-4" }, "weight": 50 },
    { "name": "treatment", "config": { "prompt_template_id": "v2", "model": "gpt-4" }, "weight": 50 }
  ],
  "metrics": ["relevance_score", "latency_p50", "user_satisfaction"],
  "primary_metric": "relevance_score",
  "guardrails": { "max_error_rate": 0.05, "max_latency_p99_ms": 3000 },
  "traffic_filter": { "product": "search", "user_segment": "premium" },
  "sample_size_target": 5000
}

Response: { "experiment_id": "exp_abc123", "status": "DRAFT" }
```

### Variant Assignment
```
GET /v1/assign?experiment_id=exp_abc123&user_id=user_456
Response: {
  "variant": "treatment",
  "config": { "prompt_template_id": "v2", "model": "gpt-4" }
}
```

### Log Outcome
```
POST /v1/outcomes
{
  "experiment_id": "exp_abc123",
  "variant": "treatment",
  "user_id": "user_456",
  "request_id": "req_789",
  "metrics": {
    "latency_ms": 450,
    "token_count": 312,
    "user_clicked_thumbs_up": true
  },
  "request_payload": { "query": "..." },
  "response_payload": { "answer": "..." }
}
```

### Offline Evaluation
```
POST /v1/eval/run
{
  "eval_suite_id": "golden_search_v3",
  "variants": [
    { "name": "v1", "config": { ... } },
    { "name": "v2", "config": { ... } }
  ],
  "scorers": ["relevance_llm_judge", "factuality_checker", "toxicity"]
}
```

## 5) Data Model (3-5 min)

### Core Entities

**Experiment**
| Column | Type | Notes |
|--------|------|-------|
| experiment_id | UUID | PK |
| name | string | |
| owner_team | string | |
| status | enum | DRAFT, RUNNING, PAUSED, CONCLUDED |
| variants | JSON | array of variant configs |
| primary_metric | string | |
| guardrails | JSON | auto-pause conditions |
| created_at | timestamp | |

**ExperimentOutcome** (high-volume event stream)
| Column | Type | Notes |
|--------|------|-------|
| outcome_id | UUID | PK |
| experiment_id | UUID | partition key |
| variant | string | |
| user_id | string | |
| request_id | string | |
| metrics | JSON | latency, scores, user signals |
| request_payload | text | stored for judge scoring |
| response_payload | text | stored for judge scoring |
| timestamp | timestamp | sort key |

**EvaluationScore**
| Column | Type | Notes |
|--------|------|-------|
| score_id | UUID | PK |
| outcome_id | UUID | FK |
| scorer_type | enum | LLM_JUDGE, HUMAN, AUTOMATED |
| dimension | string | relevance, coherence, safety |
| score | float | 0.0 - 1.0 |
| explanation | text | judge reasoning |
| scorer_model | string | model used for scoring |

**EvalSuite** (offline test datasets)
| Column | Type | Notes |
|--------|------|-------|
| suite_id | UUID | PK |
| name | string | |
| examples | JSON[] | input/expected_output pairs |
| version | int | |

### Storage Choices
- **Experiment metadata**: PostgreSQL -- relational, low volume, ACID
- **Outcomes event stream**: Kafka -- high-throughput append, consumed by scoring and analytics
- **Outcomes store**: ClickHouse -- columnar analytics, fast aggregations for metrics
- **Evaluation scores**: ClickHouse -- co-located with outcomes for join queries
- **Eval suites/datasets**: S3 + DynamoDB index -- large golden datasets stored as Parquet
- **Assignment cache**: Redis -- fast variant lookups, sticky bucketing

## 6) High-Level Architecture (5-8 min)

The platform has three main subsystems: (1) experiment management and traffic routing, (2) evaluation pipeline (automated + human scoring), and (3) statistical analysis and reporting.

### Components
- **Experiment Manager**: CRUD for experiments, lifecycle state machine, approval workflows
- **Assignment Service**: deterministic hashing for variant assignment, sticky bucketing via Redis
- **Outcome Collector**: receives metrics from LLM applications, publishes to Kafka
- **Scoring Pipeline**: consumes outcomes, runs LLM-as-judge and automated scorers
- **Human Eval Service**: samples outcomes, assigns to raters, collects structured evaluations
- **Analytics Engine**: aggregates scores per variant, computes statistical tests
- **Dashboard**: real-time experiment monitoring, metric comparison, winner declaration
- **Guardrail Monitor**: watches error rates and latency; auto-pauses experiments exceeding thresholds
- **Offline Eval Runner**: batch-runs variants against golden datasets

## 7) Deep Dive #1: Evaluation and Scoring Pipeline (8-12 min)

### Multi-Layered Scoring

**Layer 1: Automated Metrics (instant)**
- Latency, token count, error rate -- measured directly from the LLM call
- Regex-based checks (format compliance, contains required fields)
- Embedding similarity to reference answers (cosine similarity)

**Layer 2: LLM-as-Judge (seconds)**
- A separate LLM scores responses on quality dimensions
- Prompt structure: system prompt defining rubric + user message with (query, response) pair
- Output: score (1-5) + explanation
- Multiple dimensions scored independently: relevance, coherence, completeness, safety

**Judge Reliability Engineering:**
- **Position bias mitigation**: for pairwise comparisons, run both orderings and average
- **Calibration**: include known-quality examples in each batch to track judge drift
- **Cost optimization**: use a fine-tuned smaller model (Llama-3 8B) as primary judge; sample 5% to expensive model (GPT-4) for calibration
- **Caching**: cache judge scores for identical (query, response) pairs
- **Batching**: batch judge calls for throughput (32 evaluations per batch)

**Layer 3: Human Evaluation (hours)**
- Sample 3-5% of outcomes for human evaluation
- Structured rubric matching automated dimensions
- Inter-annotator agreement tracked (minimum 2 raters per item, Cohen's kappa > 0.6)
- Used to calibrate and validate LLM-as-judge accuracy

### Scoring Pipeline Architecture
1. Outcomes land in Kafka topic `experiment-outcomes`
2. Flink streaming job enriches outcomes with experiment metadata
3. Automated metrics computed inline (< 100ms)
4. LLM-judge scoring jobs consume from Kafka, call judge models, write scores back
5. Scores aggregated in ClickHouse materialized views (5-minute windows)
6. Human eval sampler picks outcomes, pushes to human eval queue

### Offline Evaluation Flow
1. User triggers eval run via API with eval suite + variants
2. Eval runner spins up parallel workers, one per variant
3. Each worker calls the LLM pipeline with test inputs
4. Responses scored by same scoring pipeline (automated + judge)
5. Results compared in a report: per-dimension scores, win rates, regressions
6. Quality gate: variant must pass offline eval before launching online experiment

## 8) Deep Dive #2: Statistical Analysis and Experiment Lifecycle (5-8 min)

### Statistical Framework
- **Primary test**: two-sample t-test for continuous metrics (mean relevance score)
- **Binary metrics**: two-proportion z-test (e.g., thumbs-up rate)
- **Multiple testing correction**: Bonferroni or Benjamini-Hochberg for multiple metrics
- **Sequential analysis**: use group-sequential design (O'Brien-Fleming boundaries) to allow early stopping while controlling false positive rate
- **Minimum sample size**: pre-computed based on desired MDE (minimum detectable effect), power (0.8), and significance level (0.05)

### Experiment Lifecycle State Machine
```
DRAFT -> REVIEW -> OFFLINE_EVAL -> RUNNING -> CONCLUDED
                                     |
                                     v
                                   PAUSED (guardrail violation)
```

1. **DRAFT**: experiment defined, not yet reviewed
2. **REVIEW**: peer review of experiment design, hypothesis, metrics
3. **OFFLINE_EVAL**: automatic offline evaluation on golden datasets as quality gate
4. **RUNNING**: live traffic split, metrics collecting
5. **PAUSED**: automatic pause if guardrails triggered (error rate, latency spike)
6. **CONCLUDED**: statistical significance reached, winner declared, results archived

### Guardrail System
- Error rate exceeds 2x control: auto-pause treatment, alert owner
- Latency p99 exceeds configured threshold: auto-pause
- Safety score drops below minimum: immediate halt
- Checked every 1 minute via streaming aggregation

### Sticky Bucketing
- Hash(user_id + experiment_id) mod 100 determines bucket
- Bucket-to-variant mapping stored in experiment config
- Ensures same user always sees same variant across sessions
- Ramp-up: change bucket ranges (e.g., 10% -> 50% -> 100%) without reassigning existing users

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| LLM-as-judge (fine-tuned small model) | Expensive frontier model for all scoring | 10x cost savings with < 5% accuracy degradation; frontier model for calibration |
| Sequential analysis | Fixed-horizon A/B test | Allows early stopping, reduces experiment duration by 30-50% |
| ClickHouse for analytics | BigQuery / Snowflake | ClickHouse handles real-time inserts + fast aggregations; lower latency for dashboards |
| Kafka for outcome streaming | Direct writes to database | Decouples collection from scoring; handles backpressure during scoring pipeline issues |
| Sticky user-level bucketing | Request-level randomization | User-level prevents inconsistent experience; request-level has more power but confuses users |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (100M requests/day, 1000 concurrent experiments)
- **Partition Kafka by experiment_id**: parallel scoring per experiment
- **Judge model fleet autoscaling**: scale GPU inference based on queue depth
- **ClickHouse sharding**: shard by experiment_id for isolated analytics
- **Assignment service caching**: pre-compute and cache assignments in edge CDN

### 100x Scale (1B requests/day)
- **Sampling**: not every request needs full scoring -- sample 10% for judge evaluation, 1% for human eval
- **Edge-based assignment**: push assignment logic to edge proxies to eliminate network hop
- **Tiered storage**: hot (last 7 days in ClickHouse), warm (30 days in Parquet on S3), cold (archive)
- **Federated experiments**: regional experiment management with global aggregation

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| Assignment service down | Fallback to control variant (safe default) |
| Judge model unavailable | Queue outcomes for retry; serve dashboards with stale data + warning |
| Kafka consumer lag | Autoscale consumers; accept delayed metric updates |
| ClickHouse query timeout | Pre-aggregate common queries in materialized views |
| Incorrect variant assignment | Deterministic hashing is stateless; any replica gives same answer |
| Bad experiment (treatment harms quality) | Guardrail monitor auto-pauses within 1 minute |

## 12) Observability and Operations (2-4 min)

### Key Metrics
- Assignment latency p99 (target: < 5ms)
- Scoring pipeline lag (target: < 5 minutes)
- Judge model agreement with human raters (target: > 85%)
- Experiment guardrail trigger rate
- Active experiment count by status and team
- Human eval queue depth and completion rate

### Alerts
- Scoring pipeline lag > 15 minutes
- Judge-human agreement drops below 75%
- Any experiment guardrail triggered
- Assignment service error rate > 0.1%

## 13) Security (1-3 min)

- **Data isolation**: experiment data partitioned by team; RBAC on experiment access
- **PII handling**: user IDs pseudonymized in outcome logs; payloads encrypted at rest
- **Judge prompt injection**: sanitize LLM responses before passing to judge to prevent prompt injection via generated content
- **Experiment integrity**: immutable experiment configs after launch (create new version instead)
- **Audit trail**: all experiment lifecycle transitions logged with actor and timestamp

## 14) Team and Operational Considerations (1-2 min)

- **Platform team (4-6 engineers)**: experiment lifecycle, assignment service, API
- **Evaluation team (3-4 engineers)**: scoring pipeline, judge model training, human eval tooling
- **Analytics team (2-3 engineers)**: statistical framework, dashboards, reporting
- **Data science partners**: help teams design experiments, review results
- **On-call**: platform on-call covers assignment service; evaluation on-call covers scoring pipeline

## 15) Common Follow-up Questions

**Q: How do you handle experiments that interact with each other?**
A: Use mutual exclusion layers -- experiments in the same layer cannot overlap traffic. Independent experiments can run in different layers simultaneously.

**Q: How do you evaluate subjective quality like "helpfulness"?**
A: Define structured rubrics with examples. Use pairwise comparison (is response A better than B?) rather than absolute scoring for subjective dimensions. Calibrate LLM judges against human pairwise preferences.

**Q: How do you handle the cost of LLM-as-judge at scale?**
A: Fine-tune a small model (8B params) as the primary judge. Calibrate against frontier model on 5% sample. Cache scores for duplicate (query, response) pairs. Sample rather than score every request.

**Q: What about long-term metrics (retention, revenue)?**
A: Experiment framework integrates with analytics warehouse. Long-term metrics are joined to experiment assignments via user_id. Holdback groups maintained for 30+ days for long-term impact measurement.

**Q: How do you prevent p-hacking?**
A: Pre-register primary metric and hypothesis before launch. Multiple testing correction for secondary metrics. Sequential analysis with spending function prevents peeking issues.

## 16) Closing Summary (30-60s)

"I've designed an LLM experimentation platform with three pillars: deterministic traffic routing with sticky bucketing, a multi-layered evaluation pipeline (automated metrics, LLM-as-judge, human raters), and rigorous statistical analysis with sequential testing and guardrails. The offline evaluation flow acts as a quality gate before live experiments. Key design choices include fine-tuned small-model judges for cost-efficiency (10x cheaper than frontier models), ClickHouse for real-time analytics, and automatic guardrails that pause experiments within 1 minute of detecting quality regressions."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| judge_sample_rate | 1.0 | 0.01 - 1.0 | Fraction of outcomes scored by LLM judge |
| human_eval_sample_rate | 0.05 | 0.01 - 0.20 | Fraction sent to human raters |
| sequential_analysis_looks | 5 | 3 - 10 | Number of interim analyses before final |
| significance_level | 0.05 | 0.01 - 0.10 | Type I error rate |
| guardrail_check_interval_sec | 60 | 30 - 300 | How often guardrails are evaluated |
| ramp_up_schedule | [10, 50, 100] | custom | Traffic percentage stages |
| metrics_aggregation_window_min | 5 | 1 - 60 | Dashboard refresh interval |
| calibration_frontier_sample_rate | 0.05 | 0.01 - 0.20 | Fraction of judge calls to expensive calibration model |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| MDE | Minimum Detectable Effect -- smallest difference worth detecting |
| Sequential Analysis | Statistical testing that allows peeking at results with controlled error rate |
| Sticky Bucketing | Ensuring same user always assigned to same variant |
| LLM-as-Judge | Using an LLM to evaluate the quality of another LLM's output |
| Pairwise Comparison | Evaluating two responses side-by-side rather than scoring individually |
| Golden Dataset | Curated test set with known-good reference answers |
| Guardrail | Automated safety check that halts experiments exceeding thresholds |
| Spending Function | Controls Type I error allocation across sequential looks |
| Inter-Annotator Agreement | Statistical measure of consistency between human raters |
| Mutual Exclusion Layer | Traffic partitioning to prevent experiment interactions |

## Appendix C: References

- "Trustworthy Online Controlled Experiments" by Kohavi, Tang, Xu
- Meta's "LLM Comparator" for pairwise evaluation
- Google's "Vertex AI Experiments" documentation
- "Judging LLM-as-a-Judge" paper (LMSYS)
- OpenAI Evals framework design
- Netflix experimentation platform architecture
- Sequential testing: O'Brien-Fleming spending functions
