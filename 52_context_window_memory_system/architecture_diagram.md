# Context Window Memory System — Architecture Diagrams

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Client
        APP[LLM Application]
    end

    subgraph Memory_Manager[Memory Manager]
        ORCHESTRATOR[Memory Orchestrator<br/>Retrieval + Assembly]
        BUDGET[Token Budget Allocator<br/>Dynamic distribution]
    end

    subgraph Short_Term[Short-Term Memory]
        REDIS_SESSION[(Redis<br/>Active Session Turns)]
        DYNAMO_HIST[(DynamoDB<br/>Persistent History)]
    end

    subgraph Working_Memory[Working Memory]
        SUMMARIZER[Summarization Service<br/>Progressive LLM summarization]
        SUMMARY_STORE[(DynamoDB<br/>Session Summaries)]
    end

    subgraph Long_Term[Long-Term Memory]
        EXTRACTOR[Memory Extractor<br/>Async fact extraction]
        DEDUP[Deduplication / Merge<br/>Semantic similarity check]
        LT_STORE[(PostgreSQL + pgvector<br/>Facts, Preferences, Entities)]
    end

    subgraph Retrieval[Retrieval Pipeline]
        EMBED_SVC[Embedding Service<br/>MiniLM-L6, 768-dim]
        VECTOR_SEARCH[Vector Search<br/>Cosine similarity + re-rank]
        RERANKER[Relevance Re-ranker<br/>Recency + frequency decay]
    end

    subgraph LLM_Layer[LLM Inference]
        CONTEXT_ASSEMBLER[Context Assembler<br/>System + memories + turns + query]
        LLM[LLM Provider]
    end

    subgraph Lifecycle[Memory Lifecycle]
        DECAY[Decay / Eviction<br/>90-day TTL]
        DELETE_API[Deletion API<br/>GDPR compliance]
        AUDIT[(Audit Log)]
    end

    APP -->|new message| ORCHESTRATOR
    ORCHESTRATOR --> BUDGET

    BUDGET -->|allocate 50%| REDIS_SESSION
    BUDGET -->|allocate 17%| SUMMARY_STORE
    BUDGET -->|allocate 25%| VECTOR_SEARCH

    REDIS_SESSION -->|recent turns| CONTEXT_ASSEMBLER
    DYNAMO_HIST -->|resumed session| REDIS_SESSION
    SUMMARY_STORE -->|session summary| CONTEXT_ASSEMBLER

    ORCHESTRATOR -->|embed query| EMBED_SVC
    EMBED_SVC --> VECTOR_SEARCH
    VECTOR_SEARCH --> LT_STORE
    VECTOR_SEARCH --> RERANKER
    RERANKER -->|top-K memories| CONTEXT_ASSEMBLER

    CONTEXT_ASSEMBLER --> LLM
    LLM -->|response| APP

    LLM -->|async| EXTRACTOR
    EXTRACTOR --> DEDUP
    DEDUP --> LT_STORE

    LLM -->|trigger if needed| SUMMARIZER
    SUMMARIZER --> SUMMARY_STORE

    DECAY --> LT_STORE
    DELETE_API --> LT_STORE
    DELETE_API --> REDIS_SESSION
    DELETE_API --> SUMMARY_STORE
    DELETE_API --> AUDIT
```

## 2. Deep-Dive: Memory Retrieval and Context Assembly

```mermaid
flowchart TB
    subgraph Input[Query Input]
        QUERY[User Message<br/>"What was that restaurant?"]
    end

    subgraph Embedding[Embedding Generation - 5ms]
        EMBED_MODEL[MiniLM-L6-v2<br/>768-dim embedding]
        QUERY_VEC[Query Vector]
    end

    subgraph LT_Retrieval[Long-Term Retrieval - 15ms]
        VECTOR_IDX[(pgvector Index<br/>User-scoped partition)]
        CANDIDATES[Top-20 Candidates<br/>Cosine similarity]
        RECENCY[Recency Boost<br/>1.2x if accessed in last hour]
        FREQUENCY[Frequency Boost<br/>1.1x if accessed > 5 times]
        CATEGORY[Category Match<br/>Boost relevant categories]
        FINAL_LT[Top-10 Long-Term Memories<br/>Fit within 3K token budget]
    end

    subgraph Session_Retrieval[Session Data - 5ms each]
        SUMMARY_FETCH[Fetch Session Summary<br/>DynamoDB lookup]
        SUMMARY_TRUNC[Truncate to 2K tokens<br/>Keep most recent segments]
        TURNS_FETCH[Fetch Recent Turns<br/>Redis LRANGE]
        TURNS_FIT[Fit turns in 6K budget<br/>Last N turns, min 3]
    end

    subgraph Assembly[Context Assembly - 2ms]
        SYS_PROMPT[System Prompt<br/>1K tokens]
        LT_BLOCK["[Memory] blocks<br/>Formatted long-term memories"]
        SUMMARY_BLOCK["[Summary] block<br/>Session summary text"]
        TURNS_BLOCK[Recent Turns<br/>Full fidelity messages]
        FINAL_CTX[Assembled Context<br/>System + LT + Summary + Turns + Query]
    end

    subgraph Budget_Tracking[Token Budget Tracking]
        COUNTER[Running Token Counter<br/>Track per-section usage]
        OVERFLOW[Overflow Handler<br/>Redistribute unused tokens]
    end

    QUERY --> EMBED_MODEL
    EMBED_MODEL --> QUERY_VEC
    QUERY_VEC --> VECTOR_IDX
    VECTOR_IDX --> CANDIDATES
    CANDIDATES --> RECENCY
    RECENCY --> FREQUENCY
    FREQUENCY --> CATEGORY
    CATEGORY --> FINAL_LT

    QUERY --> SUMMARY_FETCH
    SUMMARY_FETCH --> SUMMARY_TRUNC

    QUERY --> TURNS_FETCH
    TURNS_FETCH --> TURNS_FIT

    SYS_PROMPT --> FINAL_CTX
    FINAL_LT --> LT_BLOCK
    LT_BLOCK --> FINAL_CTX
    SUMMARY_TRUNC --> SUMMARY_BLOCK
    SUMMARY_BLOCK --> FINAL_CTX
    TURNS_FIT --> TURNS_BLOCK
    TURNS_BLOCK --> FINAL_CTX

    FINAL_LT --> COUNTER
    SUMMARY_TRUNC --> COUNTER
    TURNS_FIT --> COUNTER
    COUNTER --> OVERFLOW
    OVERFLOW -->|redistribute| TURNS_FIT
```

## 3. Critical Path Sequence: Message Processing with Memory

```mermaid
sequenceDiagram
    participant User as User
    participant App as LLM Application
    participant Orch as Memory Orchestrator
    participant Redis as Redis (Session)
    participant DDB as DynamoDB (Summary)
    participant Embed as Embedding Service
    participant PG as pgvector (Long-Term)
    participant Budget as Token Budget Allocator
    participant LLM as LLM Provider
    participant Extract as Memory Extractor
    participant Summ as Summarizer

    User->>App: "What was that restaurant I mentioned?"
    App->>Orch: Process message with memory context

    par Retrieve all memory tiers in parallel
        Orch->>Redis: LRANGE session:sess_abc (last 20 turns)
        Redis-->>Orch: 12 recent turns (7,200 tokens)

        Orch->>DDB: GET summary for sess_abc
        DDB-->>Orch: Summary text (1,800 tokens)

        Orch->>Embed: Embed query "restaurant I mentioned"
        Embed-->>Orch: 768-dim vector (5ms)
        Orch->>PG: Vector search (user_id=123, top_k=20)
        PG-->>Orch: 20 candidate memories (15ms)
    end

    Orch->>Orch: Re-rank memories by relevance + recency
    Orch->>Budget: Allocate 12K tokens across tiers
    Budget-->>Orch: LT: 3K, Summary: 2K, Turns: 6K

    Note over Orch: Select top-8 memories (2,400 tokens)<br/>Truncate summary (1,800 tokens)<br/>Last 8 turns (4,800 tokens)

    Orch->>Orch: Assemble context: system + memories + summary + turns + query
    Orch->>LLM: Send assembled context (10,200 tokens)
    LLM-->>Orch: Response: "You mentioned Osteria Francescana..."

    Orch->>Redis: RPUSH new user turn + assistant turn
    Orch-->>App: Response to user
    App-->>User: "You mentioned Osteria Francescana..."

    par Async post-processing
        Orch->>Extract: Extract new facts from last 5 turns
        Extract->>Extract: LLM call to identify memorizable facts
        Extract->>PG: Check for duplicates (cosine > 0.85)
        Extract->>PG: Store new memories with embeddings

        Note over Orch: Check if summarization needed
        alt Turn count > summarization threshold
            Orch->>Summ: Summarize turns 1-7
            Summ->>Summ: LLM call: progressive summary
            Summ->>DDB: Update session summary
            Summ->>Redis: Trim summarized turns
        end
    end
```
