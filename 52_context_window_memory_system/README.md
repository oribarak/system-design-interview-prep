# System Design Interview: Context Window Memory System — "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

"I'll design a context window memory system that enables LLM applications to maintain conversation context, recall past interactions, and manage information beyond the model's fixed context window. The system combines short-term memory (current conversation), working memory (session-level summaries), and long-term memory (persistent facts and preferences). I'll cover memory storage architectures, retrieval strategies, summarization pipelines, and how to keep context relevant while staying within token budgets."

## 1) Clarifying Questions (2-5 min)

| # | Question | Likely Answer |
|---|----------|---------------|
| 1 | What is the target LLM's context window? | 128K-200K tokens, but effective use degrades beyond 32K |
| 2 | What types of memory? Conversation history, user facts, retrieved documents? | All three: conversation turns, user profile/preferences, and RAG documents |
| 3 | How many concurrent users/sessions? | 10M active users, 1M concurrent sessions |
| 4 | How long should memory persist? | Session memory: hours; long-term memory: months/years |
| 5 | Multi-user or single-user memory? | Primarily per-user, with optional shared workspace memory |
| 6 | Is memory editable? Can users delete or correct memories? | Yes, GDPR compliance requires deletion capability |
| 7 | Privacy requirements? | Memories are per-user isolated; no cross-user leakage |
| 8 | What token budget for memory in each request? | ~8K-16K tokens reserved for memory out of 128K window |

## 2) Back-of-Envelope Estimation (3-5 min)

| Metric | Estimate |
|--------|----------|
| Active users | 10M |
| Concurrent sessions | 1M |
| Avg conversation turns per session | 15 |
| Avg tokens per turn | 200 (user) + 400 (assistant) = 600 |
| Session history (15 turns) | 9,000 tokens ~ 36 KB |
| Long-term memories per user | ~200 facts/preferences |
| Avg memory entry | 50 tokens ~ 200 bytes |
| Long-term memory per user | 200 * 200 bytes = 40 KB |
| Total long-term memory (10M users) | 10M * 40 KB = 400 TB |
| Memory reads per request | 3-5 (retrieval operations) |
| Memory read QPS | 1M sessions * 2 msgs/min / 60 = ~33K QPS |
| Summarization jobs/min | ~100K sessions needing summary updates |
| Embedding computations/sec | ~33K (for retrieval queries) |

## 3) Requirements (3-5 min)

### Functional Requirements
1. **Short-term memory**: maintain full conversation history for current session
2. **Working memory**: summarize older conversation turns to compress context
3. **Long-term memory**: extract and store persistent facts, preferences, and user profile
4. **Memory retrieval**: select the most relevant memories for each new query
5. **Token budget management**: fit all memory types within allocated token budget
6. **Memory lifecycle**: create, update, merge, expire, and delete memories
7. **User control**: users can view, correct, and delete their memories

### Non-Functional Requirements
- **Latency**: memory retrieval < 50ms (must not dominate LLM call latency)
- **Consistency**: same session always sees consistent conversation history
- **Isolation**: strict per-user memory isolation; no cross-user leakage
- **Scalability**: 10M users, 1M concurrent sessions
- **Durability**: long-term memories survive system restarts
- **Compliance**: GDPR right-to-erasure for all memory types

## 4) API Design (2-4 min)

### Memory-Augmented Completion
```
POST /v1/completions
{
  "messages": [{"role": "user", "content": "What was that restaurant I mentioned last week?"}],
  "user_id": "user_123",
  "session_id": "sess_abc",
  "memory_config": {
    "enable_short_term": true,
    "enable_long_term": true,
    "token_budget": 12000,
    "retrieval_top_k": 10
  }
}
Response: {
  "response": "You mentioned Osteria Francescana in Modena...",
  "memories_used": [
    {"type": "long_term", "content": "User visited Osteria Francescana, rated it 5 stars", "relevance": 0.94}
  ],
  "session_summary_updated": true
}
```

### Memory Management
```
GET /v1/memories?user_id=user_123&type=long_term
POST /v1/memories  { "user_id": "user_123", "content": "Prefers vegetarian options", "type": "preference" }
DELETE /v1/memories/{memory_id}
PUT /v1/memories/{memory_id}  { "content": "Updated fact..." }
```

### Session Management
```
POST /v1/sessions  { "user_id": "user_123" }
GET /v1/sessions/{session_id}/history
DELETE /v1/sessions/{session_id}
```

## 5) Data Model (3-5 min)

### Core Entities

**ConversationTurn** (short-term memory)
| Column | Type | Notes |
|--------|------|-------|
| turn_id | UUID | PK |
| session_id | UUID | partition key |
| role | enum | user, assistant, system |
| content | text | raw message |
| token_count | int | pre-computed |
| turn_index | int | ordering within session |
| created_at | timestamp | |

**SessionSummary** (working memory)
| Column | Type | Notes |
|--------|------|-------|
| session_id | UUID | PK |
| user_id | UUID | |
| summary_text | text | compressed summary of older turns |
| summary_token_count | int | |
| turns_summarized_up_to | int | turn_index watermark |
| updated_at | timestamp | |

**LongTermMemory** (persistent memory)
| Column | Type | Notes |
|--------|------|-------|
| memory_id | UUID | PK |
| user_id | UUID | partition key |
| content | text | fact, preference, or extracted entity |
| category | enum | FACT, PREFERENCE, ENTITY, EVENT |
| embedding | float[768] | for vector retrieval |
| source_session_id | UUID | provenance |
| confidence | float | extraction confidence |
| last_accessed | timestamp | for decay/eviction |
| created_at | timestamp | |

### Storage Choices
- **Conversation turns**: Redis (hot sessions) + DynamoDB (persistence) -- fast reads for active sessions, durable storage for history
- **Session summaries**: DynamoDB -- moderate volume, keyed by session_id
- **Long-term memories**: PostgreSQL + pgvector -- relational queries + vector similarity search
- **Memory embeddings index**: pgvector (< 50M memories) or Pinecone/Qdrant (> 50M)
- **Session metadata**: Redis -- fast session lookups, TTL-based expiration

## 6) High-Level Architecture (5-8 min)

When a new message arrives, the Memory Manager retrieves relevant context from three memory tiers (short-term conversation history, working session summary, long-term facts), assembles them within a token budget, and prepends them to the LLM prompt. After the LLM responds, the Memory Extractor identifies new facts to store in long-term memory and triggers summarization if the session grows too long.

### Components
- **Memory Manager**: orchestrates retrieval from all memory tiers, manages token budget
- **Short-Term Store**: conversation history for active sessions (Redis)
- **Summarizer**: compresses older conversation turns into summaries (LLM call)
- **Long-Term Store**: persistent facts and preferences with vector index
- **Memory Extractor**: post-response pipeline that identifies new facts to memorize
- **Retriever**: embeds current query and retrieves relevant long-term memories
- **Token Budget Allocator**: distributes token budget across memory types
- **Memory Lifecycle Manager**: handles expiration, merging, and user deletion requests

## 7) Deep Dive #1: Memory Retrieval and Context Assembly (8-12 min)

### Token Budget Allocation Strategy

Given a 12K token budget for memory, the allocator distributes tokens across tiers:

```
Total budget: 12,000 tokens
  System prompt:        1,000 tokens (fixed)
  Long-term memories:   3,000 tokens (25%)
  Session summary:      2,000 tokens (17%)
  Recent turns:         6,000 tokens (50%)
  -----------------------------------------
  Remaining for query:  (separate from memory budget)
```

Allocation is dynamic: if there are few long-term memories, unused tokens flow to recent turns.

### Retrieval Pipeline (< 50ms total)

**Step 1: Embed current query (5ms)**
- Use a lightweight embedding model (all-MiniLM-L6-v2 or similar)
- Compute 768-dim embedding of the user's latest message

**Step 2: Retrieve long-term memories (15ms)**
- Vector similarity search (cosine) on pgvector/Qdrant
- Top-K retrieval (K=20), then re-rank by relevance and recency
- Apply time-decay: score = similarity * decay_factor(age)
- Filter by confidence threshold (> 0.7)
- Select top 10 memories that fit within 3K token budget

**Step 3: Fetch session summary (5ms)**
- Retrieve pre-computed summary from DynamoDB
- If summary exceeds 2K tokens, truncate to most recent summary segment

**Step 4: Fetch recent turns (5ms)**
- Load last N turns from Redis (active session) or DynamoDB (resumed session)
- Include as many recent turns as fit in 6K token budget
- Always include at least the last 3 turns for coherence

**Step 5: Assemble context (2ms)**
- Order: system prompt -> long-term memories -> session summary -> recent turns -> current query
- Long-term memories formatted as: `[Memory] User prefers vegetarian restaurants.`
- Session summary formatted as: `[Previous conversation summary] ...`

### Relevance Scoring
- **Semantic similarity**: cosine distance between query embedding and memory embedding
- **Recency boost**: memories accessed in the last hour get 1.2x multiplier
- **Frequency boost**: frequently accessed memories get slight boost (likely important)
- **Category matching**: if query mentions "food", prioritize PREFERENCE memories about food

## 8) Deep Dive #2: Memory Extraction and Summarization (5-8 min)

### Memory Extraction Pipeline (asynchronous, post-response)

After each LLM response, an async pipeline analyzes the conversation to extract memorizable facts:

1. **Extraction LLM call**: prompt the LLM (cheap model) with recent turns:
   ```
   Given this conversation, extract any new facts, preferences, or important
   information about the user that should be remembered for future conversations.
   Output as JSON: [{"fact": "...", "category": "PREFERENCE|FACT|ENTITY|EVENT", "confidence": 0.0-1.0}]
   ```

2. **Deduplication**: embed extracted facts and compare against existing memories (cosine > 0.85 = duplicate)

3. **Merge**: if a new fact updates an existing memory (e.g., "moved from NYC to SF"), update rather than create new

4. **Store**: insert new memories with embeddings into long-term store

**Extraction frequency**: not every turn -- trigger extraction every 5 turns or when the conversation covers personal information (detected by keyword heuristics).

### Summarization Pipeline

When conversation history exceeds the token budget for recent turns:

1. **Trigger**: when total turns * avg_tokens > 2x budget (e.g., > 12K tokens of history)
2. **Summarize**: call LLM with turns older than the retention window:
   ```
   Summarize this conversation, preserving key decisions, questions asked,
   and any commitments made. Be concise.
   ```
3. **Progressive summarization**: new summary = summarize(old_summary + recent_turns_to_compress)
4. **Store**: update SessionSummary in DynamoDB, advance the `turns_summarized_up_to` watermark
5. **Retain**: keep last 5-8 turns in full fidelity (not summarized)

### Summarization Scheduling
- **Eager**: summarize after every 10 turns (more LLM calls, always ready)
- **Lazy**: summarize only when context assembly hits budget (fewer calls, occasional latency spike)
- **Hybrid (chosen)**: background summarization every 10 turns, but also on-demand if needed during context assembly

### Memory Decay and Eviction
- Memories have a `last_accessed` timestamp updated on each retrieval
- Memories not accessed in 90 days are candidates for archival
- Archived memories are removed from the vector index but retained in cold storage
- Users can pin memories to prevent eviction

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Alternative | Why This Choice |
|----------|-------------|-----------------|
| Progressive summarization | Full re-summarize each time | Progressive is cheaper (only summarizes delta); full re-summarize loses information over iterations |
| pgvector for memory retrieval | Pinecone/dedicated vector DB | pgvector simplifies ops (one DB for relational + vector); switch to dedicated at > 50M memories |
| Async memory extraction | Sync extraction before response | Async doesn't add latency to user-facing path; extraction can use cheap models |
| Token budget allocation (fixed ratio) | Dynamic budget based on query type | Fixed ratio is simpler and predictable; dynamic can over-allocate to wrong tier |
| Per-user memory isolation | Shared memory pool with ACLs | Isolation is simpler, prevents leakage; shared memory is complex to permission correctly |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x Scale (100M users)
- **Shard long-term memory by user_id**: pgvector partitioned by user_id hash
- **Tiered Redis**: hot sessions in Redis, warm sessions in DynamoDB with on-demand loading
- **Summarization queue**: Kafka-based async summarization with worker autoscaling
- **Embedding service**: dedicated embedding model fleet, batched inference

### 100x Scale (1B users)
- **Dedicated vector DB**: migrate from pgvector to Qdrant/Pinecone for billion-scale similarity search
- **Memory compression**: store memories as compressed embeddings; reconstruct text on retrieval via LLM
- **Hierarchical memory**: user-level summaries -> topic-level summaries -> individual memories
- **Edge caching**: pre-load active user memories at edge for < 10ms retrieval
- **Memory sharding by topic**: separate indexes for preferences, facts, events -- query only relevant shards

## 11) Reliability and Fault Tolerance (3-5 min)

| Failure Mode | Mitigation |
|--------------|------------|
| Redis (session store) down | Fall back to DynamoDB; load last N turns from persistent store |
| Vector DB down | Serve without long-term memory; log warning; recent turns still available |
| Summarization pipeline lag | Serve with full recent turns (may exceed ideal budget); queue catches up |
| Memory extraction fails | No new memories stored; next extraction cycle catches up |
| Embedding service down | Use cached embeddings for retrieval; skip new embedding computation |
| Inconsistent memory (stale summary) | Summary watermark ensures we never double-count turns |

### Data Durability
- Conversation turns written to Redis AND DynamoDB (dual write)
- Long-term memories in PostgreSQL with daily backups
- Session summaries in DynamoDB with point-in-time recovery
- Cross-region replication for disaster recovery

## 12) Observability and Operations (2-4 min)

### Key Metrics
- Memory retrieval latency p50/p99 (target: < 50ms)
- Token budget utilization (% of budget used per request)
- Memory relevance score (avg cosine similarity of retrieved memories)
- Summarization queue depth and lag
- Memory extraction rate (new memories per 1K conversations)
- Long-term memory count per user (distribution)
- Cache hit rate for frequently accessed memories

### Alerts
- Retrieval latency > 100ms p99
- Summarization queue depth > 100K
- Memory extraction error rate > 5%
- Token budget overflow rate > 1% (memories don't fit)

## 13) Security (1-3 min)

- **User isolation**: all queries scoped by user_id; no cross-user memory access
- **Encryption**: memories encrypted at rest (AES-256); in-transit (TLS)
- **Right-to-erasure**: DELETE /memories/{user_id} removes all memories, embeddings, and summaries
- **Memory audit log**: track who/what accessed or modified memories
- **PII detection**: scan extracted memories for sensitive data (SSN, credit cards); flag for review
- **No cross-session leakage**: session summaries and long-term memories kept separate; session data doesn't leak to other sessions without explicit extraction

## 14) Team and Operational Considerations (1-2 min)

- **Memory platform team (3-5 engineers)**: storage, retrieval, token budget management
- **ML team (2-3 engineers)**: embedding models, extraction prompts, summarization quality
- **Privacy/compliance (1-2 engineers)**: GDPR compliance, PII detection, deletion workflows
- **On-call**: covers storage systems (Redis, PostgreSQL), embedding service, summarization pipeline

## 15) Common Follow-up Questions

**Q: How do you handle contradictory memories?**
A: When a new memory contradicts an existing one (detected via semantic similarity + contradiction classifier), the newer memory supersedes. Old memory is soft-deleted with a link to the replacement. Users can review and correct.

**Q: How do you prevent hallucination from injected memories?**
A: Memories are clearly demarcated in the prompt with `[Memory]` tags. The system prompt instructs the LLM to treat memories as context, not absolute truth. Confidence scores help the LLM weigh uncertain memories.

**Q: What about multi-device sessions?**
A: Session state is server-side (Redis/DynamoDB), not device-dependent. User can resume any session from any device. Long-term memory is global per user.

**Q: How do you handle very long conversations (100+ turns)?**
A: Progressive summarization compresses older turns. At 100 turns, the context contains: long-term memories + summary of turns 1-90 + full turns 91-100. Summary quality degrades over many iterations, so periodic full re-summarization is scheduled.

**Q: What about shared/team memory (e.g., workspace context)?**
A: Separate memory namespace per workspace. Access controlled by workspace membership. Workspace memories are read-only for members; admins can create/edit. Workspace memories included in context with lower priority than personal memories.

## 16) Closing Summary (30-60s)

"I've designed a three-tier context memory system: short-term (full conversation turns in Redis), working memory (progressive session summaries), and long-term (persistent facts with vector retrieval from pgvector). A token budget allocator distributes 12K tokens across tiers with dynamic overflow. Memory extraction runs asynchronously post-response to identify new facts, while progressive summarization keeps session history within budget. The system scales to 10M users with per-user memory isolation and handles 33K retrieval QPS with < 50ms latency."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Range | Effect |
|------|---------|-------|--------|
| total_memory_token_budget | 12000 | 4000 - 32000 | Total tokens reserved for memory context |
| long_term_budget_pct | 0.25 | 0.10 - 0.50 | Fraction of budget for long-term memories |
| recent_turns_budget_pct | 0.50 | 0.30 - 0.70 | Fraction of budget for recent turns |
| retrieval_top_k | 20 | 5 - 50 | Number of candidate memories retrieved before re-ranking |
| similarity_threshold | 0.70 | 0.50 - 0.90 | Minimum cosine similarity for memory retrieval |
| summarization_trigger_turns | 10 | 5 - 20 | Turns between progressive summarization |
| extraction_trigger_turns | 5 | 3 - 10 | Turns between memory extraction |
| memory_decay_days | 90 | 30 - 365 | Days without access before archival |
| dedup_similarity_threshold | 0.85 | 0.75 - 0.95 | Similarity above which new memory is considered duplicate |

## Appendix B: Vocabulary Cheat Sheet

| Term | Definition |
|------|------------|
| Context Window | Maximum number of tokens an LLM can process in a single request |
| Progressive Summarization | Incrementally summarizing conversation by compressing older turns |
| Token Budget | Maximum tokens allocated for memory in each LLM request |
| Memory Extraction | Identifying and storing persistent facts from conversation turns |
| Vector Retrieval | Finding relevant memories by embedding similarity search |
| Working Memory | Session-level compressed context (summaries of older turns) |
| Memory Decay | Reducing priority of memories that haven't been accessed recently |
| Semantic Deduplication | Detecting duplicate memories via embedding similarity |
| Memory Pinning | User action to prevent a memory from being evicted |
| Conversation Affinity | Maintaining context continuity across turns in a session |

## Appendix C: References

- MemGPT: "Towards LLMs as Operating Systems" (virtual context management)
- LangChain Memory module architecture
- OpenAI memory feature design (ChatGPT persistent memory)
- pgvector extension for PostgreSQL
- "Lost in the Middle" paper (LLM attention patterns in long contexts)
- Anthropic's context window research
