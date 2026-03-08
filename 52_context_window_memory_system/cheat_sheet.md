# Context Window Memory System — Cheat Sheet

## Key Numbers
- 10M active users, 1M concurrent sessions
- 33K memory retrieval QPS
- Memory retrieval: < 50ms total (embed 5ms + vector search 15ms + session fetch 5ms + assembly 2ms)
- Token budget: 12K tokens for memory (25% long-term, 17% summary, 50% recent turns)
- Long-term memories per user: ~200 facts, ~40 KB each
- Total long-term storage: 400 TB across 10M users
- Summarization trigger: every 10 turns

## Core Components
- **Memory Orchestrator**: retrieves from all tiers in parallel, assembles context within token budget
- **Short-Term Store**: Redis for active sessions, DynamoDB for persistence
- **Summarizer**: progressive LLM-based summarization of older conversation turns
- **Long-Term Store**: PostgreSQL + pgvector for facts/preferences with vector similarity search
- **Memory Extractor**: async post-response pipeline that identifies new facts to memorize
- **Token Budget Allocator**: distributes tokens across memory tiers with dynamic overflow

## Architecture in One Sentence
Three-tier memory (short-term turns in Redis, working summaries in DynamoDB, long-term facts in pgvector) is retrieved in parallel, re-ranked by relevance and recency, and assembled within a token budget before each LLM call.

## Top 3 Trade-offs
1. **Progressive vs full re-summarization**: progressive is cheaper but accumulates information loss over many iterations
2. **pgvector vs dedicated vector DB**: pgvector simpler ops, dedicated scales better beyond 50M memories
3. **Async vs sync memory extraction**: async adds no latency but new facts aren't immediately available

## Scaling Story
- 10x: shard pgvector by user_id, tier Redis for hot/warm sessions, Kafka-based summarization queue
- 100x: dedicated vector DB, hierarchical memory (user > topic > fact), edge caching of active user memories

## Closing Statement
"Three-tier memory with parallel retrieval under 50ms. Token budget allocator ensures context stays within limits. Progressive summarization and async fact extraction keep memory current without adding user-facing latency. Per-user isolation with GDPR-compliant deletion."
