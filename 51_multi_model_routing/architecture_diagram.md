# Multi-Model Routing System — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients
        APP[Client Applications]
    end

    subgraph API_Layer[API Layer]
        GW[API Gateway<br/>Auth, Rate Limit]
    end

    subgraph Routing_Layer[Routing Layer]
        CACHE[Semantic Cache<br/>Embedding + Vector Search]
        CLASSIFIER[Complexity Classifier<br/>XGBoost, < 5ms]
        SELECTOR[Model Selector<br/>Policy + Budget + Health]
        AFFINITY[Conversation Affinity<br/>Redis Hash Map]
    end

    subgraph Policy_Config[Configuration]
        POLICY_DB[(Routing Policies<br/>PostgreSQL)]
        MODEL_REG[(Model Registry<br/>PostgreSQL + Redis)]
        BUDGET[(Budget Controller<br/>Redis Counters)]
    end

    subgraph Provider_Layer[Provider Proxy Layer]
        PROXY[Provider Proxy<br/>Retry, Failover, Circuit Breaker]
        OPENAI[OpenAI API]
        ANTHROPIC[Anthropic API]
        GOOGLE[Google API]
        SELF_HOSTED[Self-Hosted Models<br/>vLLM / TGI]
    end

    subgraph Quality_Loop[Quality Feedback Loop]
        SAMPLER[Quality Sampler<br/>5% of traffic]
        JUDGE[LLM-as-Judge<br/>Compare small vs frontier]
        RETRAIN[Classifier Retrainer<br/>Weekly batch]
    end

    subgraph Analytics[Analytics]
        KAFKA[Kafka<br/>routing-decisions topic]
        CLICKHOUSE[(ClickHouse<br/>Cost + Quality Analytics)]
        DASHBOARD[Cost & Quality Dashboard]
        HEALTH[Model Health Monitor]
    end

    APP --> GW
    GW --> CACHE
    CACHE -->|hit| GW
    CACHE -->|miss| CLASSIFIER
    CLASSIFIER --> SELECTOR
    SELECTOR --> AFFINITY
    SELECTOR --> POLICY_DB
    SELECTOR --> MODEL_REG
    SELECTOR --> BUDGET
    AFFINITY --> PROXY
    SELECTOR --> PROXY
    PROXY --> OPENAI
    PROXY --> ANTHROPIC
    PROXY --> GOOGLE
    PROXY --> SELF_HOSTED
    PROXY -->|response| GW
    PROXY -->|log| KAFKA
    GW -->|response| APP
    KAFKA --> CLICKHOUSE
    KAFKA --> SAMPLER
    SAMPLER --> JUDGE
    JUDGE --> RETRAIN
    RETRAIN -->|updated model| CLASSIFIER
    CLICKHOUSE --> DASHBOARD
    HEALTH --> MODEL_REG
    PROXY -->|health signals| HEALTH
```

## 2. Deep-Dive: Request Classification and Model Selection

```mermaid
flowchart TB
    subgraph Feature_Extraction[Feature Extraction < 2ms]
        INPUT[Incoming Request]
        TOK_COUNT[Token Counter]
        VOCAB[Vocabulary Analyzer<br/>Rare word ratio]
        TASK_DET[Task Type Detector<br/>classify, generate, reason, code]
        STRUCT[Structure Analyzer<br/>Constraint count, step markers]
        DOMAIN[Domain Detector<br/>medical, legal, general]
        CONV_DEPTH[Conversation Depth<br/>Turn count from history]
    end

    subgraph Classification[Complexity Classification < 3ms]
        FEAT_VEC[Feature Vector<br/>12 dimensions]
        XGBOOST[XGBoost Classifier<br/>~1M params, CPU inference]
        SCORE[Complexity Score<br/>0.0 - 1.0]
    end

    subgraph Selection[Model Selection < 2ms]
        FILTER_HARD[Hard Constraint Filter<br/>Context length, latency, cost cap]
        FILTER_CAP[Capability Filter<br/>coding, multilingual, reasoning]
        BUDGET_CHECK[Budget Pressure Check<br/>Redis lookup]
        TIER_MAP[Tier Mapping<br/>Score to model via policy]
        FALLBACK[Fallback Chain<br/>If primary unavailable]
    end

    subgraph Affinity_Check[Conversation Handling]
        CONV_LOOKUP[Redis Lookup<br/>conversation_id -> model]
        UPGRADE_CHECK[Complexity Upgrade Check<br/>Score > 0.8 in existing conv?]
    end

    subgraph Output[Routing Decision]
        DECISION[Selected Model + Reason]
        LOG[Log to Kafka]
    end

    INPUT --> TOK_COUNT
    INPUT --> VOCAB
    INPUT --> TASK_DET
    INPUT --> STRUCT
    INPUT --> DOMAIN
    INPUT --> CONV_DEPTH
    TOK_COUNT --> FEAT_VEC
    VOCAB --> FEAT_VEC
    TASK_DET --> FEAT_VEC
    STRUCT --> FEAT_VEC
    DOMAIN --> FEAT_VEC
    CONV_DEPTH --> FEAT_VEC
    FEAT_VEC --> XGBOOST
    XGBOOST --> SCORE
    SCORE --> FILTER_HARD
    FILTER_HARD --> FILTER_CAP
    FILTER_CAP --> BUDGET_CHECK
    BUDGET_CHECK -->|under budget| TIER_MAP
    BUDGET_CHECK -->|over budget| TIER_MAP
    TIER_MAP --> CONV_LOOKUP
    CONV_LOOKUP -->|existing conv| UPGRADE_CHECK
    CONV_LOOKUP -->|new conv| DECISION
    UPGRADE_CHECK -->|upgrade needed| DECISION
    UPGRADE_CHECK -->|keep current| DECISION
    TIER_MAP -->|primary unavailable| FALLBACK
    FALLBACK --> DECISION
    DECISION --> LOG
```

## 3. Critical Path Sequence: Request Routing and Fallback

```mermaid
sequenceDiagram
    participant Client as Client App
    participant GW as API Gateway
    participant Cache as Semantic Cache
    participant Router as Complexity Classifier
    participant Selector as Model Selector
    participant Budget as Budget Controller
    participant Health as Model Health
    participant Proxy as Provider Proxy
    participant Model as LLM Provider
    participant Fallback as Fallback Provider
    participant Log as Analytics (Kafka)

    Client->>GW: POST /v1/completions {messages, routing_policy}
    GW->>GW: Auth, rate limit check

    GW->>Cache: Compute embedding, search similar (cosine > 0.95)
    alt Cache hit
        Cache-->>GW: Cached response
        GW-->>Client: Response {cached: true, cost: $0}
    else Cache miss
        GW->>Router: Extract features, classify complexity
        Router->>Router: XGBoost inference (< 3ms)
        Router-->>GW: complexity_score = 0.35

        GW->>Selector: Select model (score=0.35, policy, constraints)
        Selector->>Budget: Check project budget remaining
        Budget-->>Selector: {remaining: $17,550, burn_rate: normal}
        Selector->>Health: Get model health status
        Health-->>Selector: {haiku: ACTIVE, sonnet: ACTIVE, opus: ACTIVE}
        Selector-->>GW: model=claude-haiku, reason="low_complexity"

        GW->>Proxy: Forward to claude-haiku
        Proxy->>Model: API call to Anthropic (Haiku)

        alt Model responds successfully
            Model-->>Proxy: Response (450ms)
            Proxy-->>GW: Response + usage metrics
            GW->>Cache: Store in semantic cache (TTL: 1hr)
            GW->>Budget: INCR spend by $0.0002
            GW->>Log: Log routing decision + cost
            GW-->>Client: Response {model: haiku, cost: $0.0002}
        else Model error / timeout
            Proxy->>Proxy: Circuit breaker check
            Proxy->>Fallback: Failover to next in chain (Sonnet)
            Fallback-->>Proxy: Response (800ms)
            Proxy-->>GW: Response from fallback
            GW->>Budget: INCR spend by $0.002
            GW->>Log: Log failover event
            GW-->>Client: Response {model: sonnet, failover: true}
        end
    end
```
