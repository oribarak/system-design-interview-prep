# Prompt Engineering Playground -- Cheat Sheet

## The #1 Thing to Understand
This is a **product-focused, client-engineer** system design. The hard part isn't the backend — it's building a responsive streaming UI that renders tokens at 60fps, handles side-by-side model comparison, and provides git-like prompt versioning for iterative development.

## Key Numbers
- 50K DAU, ~30 QPS prompt executions at peak
- Tokens stream at ~50 tokens/s = one token every 20ms
- 60fps render budget = 16ms per frame (multiple tokens batch into one frame)
- Prompt version: ~5 KB each. 25M versions = 125 GB in PostgreSQL
- Auto-save drafts to Redis every 5s
- TTFT target: < 1s

## The Three Mechanisms That Matter
1. **requestAnimationFrame buffered rendering**: Tokens accumulate in a mutable buffer, flushed to DOM once per frame. Bypasses React reconciliation on the hot path. Prevents UI jank at high token rates.
2. **Git-like prompt versioning**: Each save creates an immutable PromptVersion snapshot. Auto-version on every "Run" for reproducibility. JSON Patch diffs between versions. Content-addressable for deduplication.
3. **SSE-based comparison multiplexing**: Two parallel inference streams tagged with stream_id, rendered independently in split panes. Comparison metrics (tokens, latency, cost) shown after both complete.

## Top 3 Trade-offs
1. **React state per token vs. buffered DOM**: Buffered DOM gives 60fps. React state per token causes jank at 50 tokens/s.
2. **Full version snapshots vs. delta chains**: Snapshots are simpler and cheap (5 KB each). Deltas save space but complicate reads.
3. **SSE vs. WebSocket for streaming**: SSE is simpler and unidirectional (sufficient). WebSocket needed only if tool results flow back mid-stream (use SSE + POST for tool results).

## Tool Call Handling
When LLM emits tool_use: pause stream -> render tool call card -> user provides mock result -> result sent back -> generation continues. Three modes: manual, auto (mock server), simulated (small model generates result).

## Scaling Story
- 1x: Single streaming proxy, PostgreSQL for versions, Redis for drafts
- 10x: Horizontal streaming proxies, Redis Cluster, CDN for SPA, read replicas
- 100x: Shard by workspace, regional streaming proxies, prompt template marketplace

## Closing Statement
"A product-focused design centered on streaming UX and iterative workflow. requestAnimationFrame buffering renders tokens at 60fps. Git-like versioning with immutable snapshots enables prompt iteration tracking. Side-by-side comparison runs parallel inference streams. Monaco editor handles large prompts. Scales via stateless streaming proxies and CDN."
