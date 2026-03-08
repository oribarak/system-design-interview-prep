# Context Window Memory System -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a system that lets LLM applications maintain conversation context and recall past interactions beyond the model's fixed context window. The hard part is managing three tiers of memory (recent turns, session summaries, long-term facts), fitting them within a token budget, and retrieving the most relevant memories in under 50ms without degrading the user experience.

## 2. Requirements
**Functional:** Maintain full conversation history for active sessions. Progressively summarize older turns to compress context. Extract and store persistent facts and preferences from conversations. Retrieve the most relevant memories for each new query. Manage token budget allocation across memory types. Let users view, correct, and delete their memories.

**Non-functional:** Memory retrieval under 50ms. Strict per-user isolation with no cross-user leakage. Support 10M users and 1M concurrent sessions. GDPR right-to-erasure. Long-term memories survive restarts.

## 3. Core Concept: Three-Tier Memory with Token Budget Allocation
Memory is organized into three tiers that fit within a fixed token budget (e.g., 12K tokens). Recent turns (50% of budget) preserve the last 5-8 messages in full fidelity. Session summaries (17%) compress older conversation turns via progressive summarization. Long-term memories (25%) store persistent facts and preferences retrieved via vector similarity search. A remaining 8% goes to the system prompt. When one tier underflows, unused tokens flow to others. This approach keeps context relevant while staying within the model's effective attention range.

## 4. High-Level Architecture
```
New Message --> [Memory Manager] --> assembles context within token budget
                     |
      [Short-Term Store] -- Redis (full recent turns)
      [Session Summary] -- DynamoDB (compressed older turns)
      [Long-Term Store] -- pgvector (facts + preferences with embeddings)
                     |
              [LLM Inference] --> response
                     |
              [Memory Extractor] -- async: identifies new facts to store
              [Summarizer] -- async: compresses older turns when needed
```
- **Memory Manager**: Orchestrates retrieval from all tiers, manages token budget allocation.
- **Retriever**: Embeds current query, performs vector similarity search on long-term memories with recency and relevance scoring.
- **Memory Extractor**: Async post-response pipeline using a cheap LLM to identify new facts from conversation.
- **Summarizer**: Progressive summarization -- compresses old_summary + recent_turns_to_compress.

## 5. How Memory Retrieval Works
1. User sends a new message.
2. Memory Manager embeds the query using a lightweight embedding model (5ms).
3. Vector similarity search retrieves top-20 long-term memories from pgvector, re-ranked by relevance and recency (15ms).
4. Session summary is fetched from DynamoDB (5ms).
5. Recent turns are loaded from Redis (5ms).
6. Context is assembled: system prompt, long-term memories, session summary, recent turns, current query.
7. After the LLM responds, the Memory Extractor asynchronously identifies new facts to store and the Summarizer compresses turns if history exceeds the budget.

## 6. What Happens When Things Fail?
- **Vector DB down**: Serve without long-term memory. Recent turns and session summary still provide context. Log a warning.
- **Redis (session store) down**: Fall back to DynamoDB for recent turns. Slightly higher latency but no data loss.
- **Summarization pipeline lag**: Serve with full recent turns even if they exceed the ideal budget. Queue catches up in the background.

## 7. Scaling
- **10x**: Shard long-term memory by user_id. Tiered Redis with hot sessions in memory, warm sessions in DynamoDB. Kafka-based async summarization with worker autoscaling.
- **100x**: Migrate from pgvector to dedicated vector DB (Qdrant/Pinecone) for billion-scale similarity search. Hierarchical memory with user-level, then topic-level summaries. Edge caching of active user memories.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Progressive summarization vs full re-summarize | Progressive is cheaper (only summarizes delta) but quality degrades over many iterations |
| pgvector vs dedicated vector DB | pgvector simplifies ops (one DB for relational + vector); switch to dedicated at 50M+ memories |
| Async memory extraction | Doesn't add latency to user-facing path but means new facts aren't available until the next turn |

## 9. Closing (30s)
> This is a three-tier memory system: short-term (full turns in Redis), working memory (progressive session summaries), and long-term (persistent facts with vector retrieval from pgvector). A token budget allocator distributes 12K tokens across tiers with dynamic overflow. Memory extraction runs asynchronously to identify new facts, while progressive summarization keeps history within budget. The key insight is that 50% of the budget goes to recent turns for conversation coherence, while vector retrieval ensures relevant long-term facts surface when needed.
