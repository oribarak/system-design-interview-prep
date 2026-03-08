# Experimentation & Evaluation Platform for LLMs — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients
        APP[LLM Application]
        DS[Data Scientist / PM]
    end

    subgraph Experiment_Management[Experiment Management]
        EXP_API[Experiment API]
        EXP_DB[(Experiment DB<br/>PostgreSQL)]
        LIFECYCLE[Lifecycle Manager<br/>State Machine]
    end

    subgraph Traffic_Routing[Traffic Routing]
        ASSIGN[Assignment Service]
        ASSIGN_CACHE[(Assignment Cache<br/>Redis)]
    end

    subgraph Data_Collection[Data Collection]
        COLLECTOR[Outcome Collector]
        KAFKA[Kafka<br/>experiment-outcomes topic]
    end

    subgraph Evaluation_Pipeline[Evaluation Pipeline]
        AUTO_SCORER[Automated Scorer<br/>Latency, Format, Embedding Similarity]
        JUDGE[LLM-as-Judge Service<br/>Fine-tuned 8B Model]
        JUDGE_CAL[Calibration Judge<br/>Frontier Model, 5% sample]
        HUMAN_SAMPLER[Human Eval Sampler]
        HUMAN_UI[Human Evaluation UI]
    end

    subgraph Offline_Eval[Offline Evaluation]
        EVAL_RUNNER[Eval Runner<br/>Batch Workers]
        GOLDEN_DS[(Golden Datasets<br/>S3 + DynamoDB)]
    end

    subgraph Analytics[Analytics & Reporting]
        ANALYTICS_ENGINE[Analytics Engine<br/>Statistical Tests]
        CLICKHOUSE[(ClickHouse<br/>Outcomes + Scores)]
        DASHBOARD[Experiment Dashboard]
        GUARDRAIL[Guardrail Monitor]
    end

    subgraph ML_Infra[Judge Model Infra]
        GPU_FLEET[GPU Fleet<br/>Judge Models]
        MODEL_REG[Model Registry]
    end

    DS -->|create experiment| EXP_API
    EXP_API --> EXP_DB
    EXP_API --> LIFECYCLE
    APP -->|get variant| ASSIGN
    ASSIGN --> ASSIGN_CACHE
    ASSIGN -->|lookup config| EXP_DB
    ASSIGN -->|variant config| APP
    APP -->|log outcome| COLLECTOR
    COLLECTOR --> KAFKA
    KAFKA --> AUTO_SCORER
    KAFKA --> JUDGE
    KAFKA --> HUMAN_SAMPLER
    AUTO_SCORER -->|scores| CLICKHOUSE
    JUDGE -->|inference| GPU_FLEET
    JUDGE -->|scores| CLICKHOUSE
    JUDGE -->|5% sample| JUDGE_CAL
    JUDGE_CAL -->|calibration scores| CLICKHOUSE
    HUMAN_SAMPLER --> HUMAN_UI
    HUMAN_UI -->|scores| CLICKHOUSE
    CLICKHOUSE --> ANALYTICS_ENGINE
    ANALYTICS_ENGINE --> DASHBOARD
    CLICKHOUSE --> GUARDRAIL
    GUARDRAIL -->|auto-pause| LIFECYCLE
    DS -->|view results| DASHBOARD
    LIFECYCLE -->|trigger offline eval| EVAL_RUNNER
    EVAL_RUNNER --> GOLDEN_DS
    EVAL_RUNNER -->|results| CLICKHOUSE
    GPU_FLEET --> MODEL_REG
```

## 2. Deep-Dive: LLM-as-Judge Scoring Pipeline

```mermaid
flowchart TB
    subgraph Input[Outcome Ingestion]
        KAFKA_IN[Kafka Consumer Group<br/>Partitioned by experiment_id]
        DEDUP[Deduplication Filter<br/>Check outcome_id]
    end

    subgraph Enrichment[Enrichment]
        ENRICH[Metadata Enricher<br/>Add experiment config, rubric]
        RUBRIC_STORE[(Rubric Store<br/>Scoring dimensions per experiment)]
    end

    subgraph Judge_Pipeline[Judge Execution]
        BATCHER[Request Batcher<br/>32 items per batch]
        PROMPT_BUILDER[Judge Prompt Builder<br/>Rubric + Query + Response]
        JUDGE_MODEL[Primary Judge Model<br/>Fine-tuned Llama-3 8B]
        CALIBRATION[Calibration Router<br/>5% to frontier model]
        FRONTIER[Frontier Judge<br/>GPT-4 class model]
        SCORE_PARSER[Score Parser<br/>Extract score + explanation]
    end

    subgraph Quality_Control[Judge Quality Control]
        CACHE_CHECK[Score Cache<br/>Hash of query+response]
        KNOWN_EXAMPLES[Calibration Examples<br/>Known-quality canaries]
        DRIFT_DETECTOR[Judge Drift Detector<br/>Compare canary scores over time]
        POSITION_SWAP[Position Bias Mitigator<br/>Swap order for pairwise]
    end

    subgraph Output[Score Storage]
        SCORE_WRITER[Score Writer]
        CLICKHOUSE_OUT[(ClickHouse<br/>evaluation_scores table)]
        METRICS[Prometheus Metrics<br/>Judge latency, agreement rate]
    end

    KAFKA_IN --> DEDUP
    DEDUP --> ENRICH
    ENRICH --> RUBRIC_STORE
    ENRICH --> CACHE_CHECK
    CACHE_CHECK -->|cache hit| SCORE_WRITER
    CACHE_CHECK -->|cache miss| BATCHER
    BATCHER --> PROMPT_BUILDER
    PROMPT_BUILDER -->|inject canary| KNOWN_EXAMPLES
    PROMPT_BUILDER --> JUDGE_MODEL
    PROMPT_BUILDER --> CALIBRATION
    CALIBRATION -->|5%| FRONTIER
    CALIBRATION -->|95%| JUDGE_MODEL
    JUDGE_MODEL --> SCORE_PARSER
    FRONTIER --> SCORE_PARSER
    SCORE_PARSER --> POSITION_SWAP
    POSITION_SWAP --> SCORE_WRITER
    KNOWN_EXAMPLES --> DRIFT_DETECTOR
    DRIFT_DETECTOR --> METRICS
    SCORE_WRITER --> CLICKHOUSE_OUT
    SCORE_WRITER --> METRICS
```

## 3. Critical Path Sequence: Online Experiment Lifecycle

```mermaid
sequenceDiagram
    participant DS as Data Scientist
    participant API as Experiment API
    participant LM as Lifecycle Manager
    participant Eval as Offline Eval Runner
    participant Assign as Assignment Service
    participant App as LLM Application
    participant Collect as Outcome Collector
    participant Judge as Judge Pipeline
    participant Stats as Analytics Engine
    participant Guard as Guardrail Monitor

    DS->>API: POST /experiments {variants, metrics, guardrails}
    API->>API: Validate config, create experiment
    API-->>DS: {experiment_id, status: DRAFT}

    DS->>API: POST /experiments/{id}/submit_review
    API->>LM: Transition DRAFT -> REVIEW
    LM-->>DS: Review approved

    LM->>Eval: Trigger offline evaluation
    Eval->>Eval: Run variants against golden dataset
    Eval->>Judge: Score responses
    Judge-->>Eval: Scores per variant per dimension
    Eval-->>LM: Offline eval report (PASS/FAIL)

    alt Offline eval passes
        LM->>LM: Transition to RUNNING (10% ramp)
        LM->>Assign: Register experiment + variant mapping

        loop For each user request
            App->>Assign: GET /assign?experiment_id&user_id
            Assign->>Assign: Hash(user_id + exp_id) mod 100
            Assign-->>App: {variant: "treatment", config: {...}}
            App->>App: Execute LLM pipeline with variant config
            App->>Collect: POST /outcomes {metrics, payloads}
            Collect->>Judge: Async scoring via Kafka
        end

        loop Every 1 minute
            Guard->>Guard: Check error rate, latency, safety scores
            alt Guardrail violated
                Guard->>LM: Auto-pause experiment
                LM-->>DS: Alert: experiment paused
            end
        end

        loop Every 5 minutes
            Stats->>Stats: Aggregate scores per variant
            Stats->>Stats: Sequential analysis (O'Brien-Fleming)
            alt Significance reached
                Stats-->>DS: Winner declared with confidence interval
            end
        end

        DS->>API: POST /experiments/{id}/conclude
        API->>LM: Transition RUNNING -> CONCLUDED
        LM->>LM: Archive results, generate report
    else Offline eval fails
        LM-->>DS: Blocked: offline eval regression detected
    end
```
