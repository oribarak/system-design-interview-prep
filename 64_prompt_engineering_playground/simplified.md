# Prompt Engineering Playground -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
Design a prompt engineering playground (like Anthropic Console or OpenAI Playground) where developers interactively craft, test, and iterate on prompts. This is a product-focused, client-engineer design. The hard parts: rendering a high-frequency token stream at 60fps, supporting side-by-side model comparison, and providing git-like version history for prompt iterations.

## 2. Requirements
**Functional:** Compose prompts with system messages, multi-turn conversations, parameters (temperature, max_tokens), and tool definitions. Stream LLM responses token-by-token. Side-by-side comparison (same prompt, different models/params). Version history with diffs. Team workspaces. Export as code snippets.

**Non-functional:** TTFT < 1s. Editor operations < 200ms. Support 200K token context windows without UI freezing. 99.9% availability.

## 3. High-Level Architecture
```
Developer --> [React SPA (Monaco Editor + Output Pane)]
                        |
                   [API Gateway] --> [Streaming Proxy] --> [Inference API (GPUs)]
                        |                    |
                   [Playground API]    [Comparison Engine]
                        |              (parallel streams)
                  [Version Store]
                  (PostgreSQL)
                        |
                   [Redis] --------- [ClickHouse]
                  (drafts)          (run analytics)
```
- **React SPA**: Split-pane layout. Monaco editor for prompts, output viewer for streaming results.
- **Streaming Proxy**: Proxies SSE connections to inference API. Handles comparison multiplexing.
- **Version Store**: Git-like immutable prompt version snapshots.

## 4. Deep Dive: Token Streaming at 60fps

Tokens arrive at ~50/second (every 20ms). Naive approach: setState per token = React re-renders 50x/second = UI jank.

**Fix: requestAnimationFrame buffering.**
- Tokens accumulate in a mutable buffer (no React state update).
- On each animation frame (~16ms), flush all buffered tokens to the DOM directly.
- React state only updates for metadata (token count, cost).
- Multiple tokens batch into one frame — React reconciliation avoided on the hot path.

**For large outputs (>10K tokens):** Virtualized rendering — only render visible lines. Full text kept in a plain string buffer.

**For code blocks:** Incremental syntax highlighting (Tree-sitter/Lezer) — only highlight new tokens, not the entire output.

## 5. Deep Dive: Prompt Versioning

Each prompt has a version timeline:
- **Auto-save**: Drafts saved to Redis every 5s (crash recovery, not a version).
- **Explicit save**: Creates an immutable `PromptVersion` in PostgreSQL. Optional commit message.
- **Auto-version on run**: Every time the user clicks "Run" with a changed config, a version is auto-created. Every run is reproducible.

**Diffs:** JSON Patch (RFC 6902) between versions. UI shows parameter changes (red/green), system prompt diffs (inline), message list changes.

**Version timeline UI:** Vertical timeline in the side panel. Click any version to load. Compare any two versions side-by-side. Fork from any historical version.

## 6. Side-by-Side Comparison

User selects two model configs, clicks "Compare". Two SSE streams open in parallel:
- Each stream renders independently in its own pane.
- Faster stream shows "waiting for other model" after completing.
- On both completion: comparison metrics (token count, latency, cost).

Multiplexing: single SSE connection with stream_id tags, or two separate connections.

## 7. Tool Call Handling

When the LLM emits a `tool_use` block mid-stream:
1. Streaming pauses.
2. UI shows a tool call card (function name, arguments).
3. User provides a mock result (or auto-execute against a mock server).
4. Result sent back, generation continues.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| requestAnimationFrame buffering vs. React state per token | 60fps vs. janky. Buffering bypasses React reconciliation. |
| Full version snapshots vs. delta chains | Snapshots simpler, cheap at 5 KB each. Deltas save space but complicate reads. |
| SSE vs. WebSocket | SSE is simpler and sufficient for unidirectional streaming. |
| Monaco vs. textarea | Monaco handles large text, syntax highlighting, but heavier to load. |

## 9. Scaling
- **10x**: Stateless streaming proxies scale horizontally. Redis Cluster for drafts. CDN for the SPA.
- **100x**: Shard prompt storage by workspace. Regional streaming proxies. Rate limiting per workspace for expensive model runs.

## 10. Closing (30s)
> The key challenge is streaming UX: tokens arrive at 50/second and naive React rendering janks. requestAnimationFrame buffering flushes tokens to the DOM once per frame, bypassing React reconciliation. Git-like versioning stores immutable prompt snapshots with auto-versioning on every run for reproducibility. Side-by-side comparison runs parallel inference streams with synchronized rendering. The architecture is a React SPA with Monaco editor, a stateless streaming proxy for SSE, and PostgreSQL-backed versioning.
