# System Design Interview: Real-Time AI Coding Assistant (Claude Code) -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a real-time AI coding assistant -- think Claude Code -- that accepts natural language instructions, streams LLM-generated responses token-by-token, and can autonomously execute tools (file reads, edits, shell commands, web searches) in a multi-turn agentic loop. The key challenges are low-latency streaming from GPU inference, orchestrating tool calls within a conversation context, managing long context windows that grow across turns, and keeping the system safe and responsive under load. I will walk through the end-to-end architecture from CLI client to GPU backend, then deep-dive into the agentic tool execution loop and the streaming/context management layer."

---

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | Is this a CLI tool, IDE extension, or web-based? | Primarily CLI, but the backend should support any client. |
| 2 | What models do we serve (single model or multi-model routing)? | Multiple models (e.g., Haiku, Sonnet, Opus) with user-selectable or auto-routed tiers. |
| 3 | Do we need agentic tool use (file I/O, shell, web)? | Yes -- the assistant reads/writes files, runs commands, and searches the web autonomously. |
| 4 | What is the target latency for time-to-first-token? | < 500ms for interactive, < 2s for complex tool-planning turns. |
| 5 | How long can conversations get? | Hundreds of turns; context can reach 100K-200K tokens. |
| 6 | Do we need multi-user / team support? | Yes -- usage tracking, rate limiting, and permission controls per user/org. |
| 7 | What safety requirements exist? | Tool calls must be sandboxed or user-approved; no arbitrary code execution without consent. |

---

## 2) Back-of-Envelope Estimation (3-5 min)

**Traffic:**
- 1M active users, average 20 sessions/day, 5 turns/session = 100M turns/day.
- Average input context: 2,000 tokens (instruction + file contents + conversation history).
- Average output: 500 tokens.
- Peak QPS: 100M / 86400 * 3 (burst) = ~3,500 QPS.

**Compute:**
- Output tokens/day: 100M * 500 = 50B tokens/day.
- Peak output token rate: ~1.7M tokens/s.
- At 50 tokens/s per GPU (Opus-class, batched): 1.7M / 50 = ~34,000 GPU-equivalents.
- With tensor parallelism (TP=8 for largest model): ~4,250 nodes.

**Storage:**
- Conversation logs: 100M turns * 2,500 tokens * 4 bytes = ~1 TB/day.
- Tool execution logs (commands, outputs): ~500 GB/day.
- Model weights in memory: 140-400 GB per model copy (FP16).

**Bandwidth:**
- Streaming output: 3,500 QPS * 500 tokens * 5 bytes/token = ~8.75 MB/s (modest).
- Tool result upload (file contents): up to 100 KB per tool call, ~10K tool calls/s peak = ~1 GB/s.

---

## 3) Requirements (3-5 min)

### Functional
- FR1: Accept natural language prompts and stream LLM-generated responses token-by-token.
- FR2: Execute tool calls (file read/write/edit, shell commands, web search, glob/grep) within the conversation loop.
- FR3: Support multi-turn conversations with growing context.
- FR4: Allow user to approve/deny tool calls based on permission settings.
- FR5: Support model selection (fast mode vs. full mode) and mid-conversation switching.
- FR6: Maintain conversation history with automatic context compression when approaching limits.

### Non-functional
- NF1: **Low latency** -- time-to-first-token < 500ms for cached contexts.
- NF2: **Streaming** -- inter-token latency < 50ms for smooth UX.
- NF3: **Availability** -- 99.9% uptime; graceful degradation under load.
- NF4: **Safety** -- sandboxed tool execution; user permission gates; no data exfiltration.
- NF5: **Scalability** -- handle 3,500+ concurrent sessions at peak.
- NF6: **Cost efficiency** -- route simple queries to smaller models; use prompt caching.

---

## 4) API Design (2-4 min)

### Client-to-Backend API (WebSocket + REST hybrid)

```
# Start or resume a conversation
POST /v1/conversations
  Body: { model: "opus-4", system_prompt: string, tools: ToolConfig[] }
  Returns: { conversation_id: string, websocket_url: string }

# Send a user turn (over WebSocket)
WS SEND: {
  type: "user_message",
  content: string,
  attachments: [{ type: "file", path: string, content: string }]
}

# Receive streaming assistant response (over WebSocket)
WS RECV: { type: "text_delta", delta: string }
WS RECV: { type: "tool_use", tool: string, params: object, request_id: string }
WS RECV: { type: "turn_complete", usage: { input_tokens, output_tokens } }

# Submit tool result
WS RECV (from client): {
  type: "tool_result",
  request_id: string,
  result: string | { error: string }
}

# Model/config update mid-conversation
POST /v1/conversations/{id}/config
  Body: { model: "haiku-4", permission_mode: "auto" }
```

---

## 5) Data Model (3-5 min)

### Core Entities

**Conversation**
| Column | Type | Notes |
|--------|------|-------|
| conversation_id | UUID | PK |
| user_id | UUID | FK, shard key |
| org_id | UUID | For billing/limits |
| model | string | Current model selection |
| system_prompt | text | System instructions |
| created_at | timestamp | |
| last_active_at | timestamp | TTL for cleanup |

**Turn**
| Column | Type | Notes |
|--------|------|-------|
| turn_id | UUID | PK |
| conversation_id | UUID | FK |
| role | enum | user / assistant / tool_result |
| content | text | Raw content or compressed summary |
| tool_calls | jsonb | Array of tool invocations |
| token_count | int | For context budget tracking |
| created_at | timestamp | |

**ToolExecution**
| Column | Type | Notes |
|--------|------|-------|
| execution_id | UUID | PK |
| turn_id | UUID | FK |
| tool_name | string | e.g., "bash", "read", "edit" |
| params | jsonb | Tool input parameters |
| result | text | Tool output (truncated if large) |
| status | enum | pending / approved / denied / completed / error |
| duration_ms | int | Execution time |

**Storage choices:**
- PostgreSQL for conversations/turns (strong consistency for user data).
- Shard by user_id for even distribution.
- Object store (S3) for large tool outputs and conversation archives.
- Redis for active session state and prompt cache keys.

---

## 6) High-Level Architecture (5-8 min)

### Dataflow
1. CLI client connects via WebSocket to the **Gateway**.
2. Gateway authenticates, rate-limits, and routes to a **Session Manager**.
3. Session Manager assembles the full prompt (system prompt + conversation history + tool results) and sends to the **Inference Router**.
4. Inference Router selects the optimal GPU pool based on model, load, and prompt cache locality, forwarding to the **Inference Engine**.
5. Inference Engine runs the LLM, streaming tokens back through the chain.
6. When the LLM emits a tool_use block, the Session Manager pauses streaming and sends the tool request to the client.
7. Client executes the tool locally (with permission check), returns the result.
8. Session Manager appends the tool result to context and re-invokes the Inference Engine for the next assistant turn.
9. This loop repeats until the assistant produces a final text response with no tool calls.

### Components
- **CLI Client**: Local process that renders streaming text, manages file system access, executes sandboxed tools, handles permission prompts.
- **API Gateway**: TLS termination, auth (API key / OAuth), rate limiting, WebSocket upgrade, request routing.
- **Session Manager**: Stateful service that owns conversation context assembly, tool call orchestration, context compression, and the agentic loop.
- **Inference Router**: Selects the best inference pool based on model, prompt cache hit probability, queue depth, and cost tier.
- **Inference Engine**: GPU cluster running vLLM/TensorRT-LLM with continuous batching, KV-cache management, and LoRA adapter switching.
- **Prompt Cache**: Distributed cache (Redis + GPU KV-cache) for prefix caching -- reuses computation for repeated system prompts and conversation prefixes.
- **Context Compressor**: Summarizes older turns when context approaches the window limit, preserving key information while freeing token budget.
- **Usage & Billing Service**: Tracks token consumption per user/org, enforces quotas, emits billing events.
- **Logging Pipeline**: Captures prompts, completions, tool calls, and latencies for observability and safety review.

### One-sentence summary
> "A WebSocket-connected agentic loop where the Session Manager orchestrates multi-turn LLM inference and client-side tool execution, backed by a GPU inference cluster with prompt caching and continuous batching."

---

## 7) Deep Dive #1: Agentic Tool Execution Loop (8-12 min)

### The Core Loop

The key differentiator of an AI coding assistant is the **agentic loop**: the LLM autonomously decides which tools to call, processes results, and continues reasoning until the task is complete.

#### Loop State Machine
```
IDLE -> INFERRING -> [TEXT_COMPLETE | TOOL_REQUESTED]
TOOL_REQUESTED -> AWAITING_PERMISSION -> [APPROVED -> EXECUTING | DENIED -> INFERRING]
EXECUTING -> TOOL_COMPLETE -> INFERRING
TEXT_COMPLETE -> IDLE
```

#### Key Design Decisions

**1. Tool execution location: client-side vs server-side**
- File I/O and shell commands run **client-side** -- the CLI has direct filesystem access and the user can see/approve actions.
- Web searches and MCP server calls can run **server-side** for lower latency.
- This hybrid approach avoids shipping the user's entire codebase to the server.

**2. Permission model**
- Three modes: `ask` (prompt user for each tool), `auto` (allow all within a category), `deny` (block tool category).
- Permission decisions are made client-side with zero server round-trips.
- Dangerous operations (git push, file delete) always require explicit approval regardless of mode.

**3. Parallel tool calls**
- The LLM can emit multiple tool_use blocks in a single turn.
- Independent tools execute in parallel on the client (e.g., reading 3 files simultaneously).
- Dependent tools execute sequentially (e.g., glob then read).
- The Session Manager collects all results before the next inference call.

**4. Context growth management**
- Each tool result adds tokens to the context. A file read can add 2,000+ tokens.
- Strategy: truncate large tool outputs (cap at 2,000 lines), summarize after N turns.
- The Context Compressor runs when context exceeds 80% of the model's window.

**5. Error handling and recovery**
- Tool timeout: 2 minutes for shell commands, 30s for file operations.
- On tool error: include the error message in context so the LLM can adapt.
- On inference error: retry with exponential backoff; fall back to a smaller model if the primary is overloaded.
- On client disconnect: save session state; allow resume within 30 minutes.

#### Loop Latency Budget
| Phase | Target | Notes |
|-------|--------|-------|
| Context assembly | < 50ms | Cached prefix lookup |
| Inference TTFT | < 500ms | With prompt caching |
| Token streaming | < 50ms/token | Inter-token latency |
| Tool execution (local) | < 5s typical | Depends on operation |
| Tool result ingestion | < 20ms | Append to context |

---

## 8) Deep Dive #2: Streaming and Prompt Caching (5-8 min)

### Token Streaming Architecture

**Server-Sent Events over WebSocket:**
- The Inference Engine produces tokens one at a time via continuous batching.
- Tokens are pushed to the Session Manager, which forwards them over the WebSocket to the client.
- The client renders tokens incrementally (character-by-character for text, buffered for code blocks).
- Backpressure: if the client can't keep up, the WebSocket buffer grows; the server pauses streaming at 1MB buffer depth.

### Prompt Caching (Critical for Latency)

In a multi-turn conversation, 80-90% of the prompt is identical between turns (system prompt + prior conversation). Re-computing attention for all prior tokens is wasteful.

**Prefix caching strategy:**
1. Hash the prompt prefix (system prompt + turns 1..N-1).
2. Check if the KV-cache for this prefix exists on any GPU in the pool.
3. If hit: route the request to that GPU; only compute attention for new tokens.
4. If miss: compute full attention but store the KV-cache for future turns.

**Cache eviction:**
- LRU eviction on GPU memory (KV-cache competes with batch size).
- Pin system prompts that are shared across many users (e.g., Claude Code's default prompt).
- Time-based eviction: drop caches for sessions idle > 5 minutes.

**Impact:**
- Cache hit reduces TTFT from ~2s to ~300ms for a 50K-token context.
- Cache hit rate in practice: 70-85% for active conversations.
- GPU memory trade-off: dedicating 30% of HBM to KV-cache reduces max batch size by ~20%.

### Context Window Compression

When context exceeds 80% of the window:
1. Identify turns older than the last 10 that contain no referenced file paths or active state.
2. Summarize those turns into a compact block (~500 tokens for 20 turns).
3. Replace original turns with the summary block.
4. Invalidate the KV-cache prefix (next turn will be a cache miss).

---

## 9) Trade-offs and Alternatives (3-5 min)

### Server-side vs Client-side Tool Execution
| Factor | Client-side | Server-side |
|--------|-------------|-------------|
| Security | User controls their files | Server needs file access |
| Latency | No upload needed | Extra round-trip for file content |
| Scalability | Offloads compute to client | Server handles everything |
| Offline | Works without internet for tools | Requires connectivity |
**Decision:** Client-side for file/shell ops, server-side for web/API calls.

### WebSocket vs SSE vs HTTP Long-Polling
- **WebSocket** chosen for bidirectional communication (tool results flow back).
- SSE is simpler but unidirectional -- requires a separate POST channel for tool results.
- Long-polling adds latency and complexity.

### Single Large Model vs Multi-Model Routing
- Simple queries (explain code, rename variable) routed to Haiku (cheaper, faster).
- Complex tasks (architect a system, debug race conditions) routed to Opus.
- Routing decision made by a lightweight classifier or user toggle (/fast mode).
- Trade-off: routing errors cause user frustration vs. cost savings of 5-10x on simple queries.

### CAP/PACELC
- Conversation state is per-user; strong consistency within a session is required (no stale turns).
- Across sessions: eventual consistency is acceptable (usage counters, billing).
- During partition: prefer availability -- serve from cached model weights; queue billing updates.

---

## 10) Scaling: 10x and 100x (3-5 min)

### At 10x (35K QPS, 10M users)
- **Inference cluster**: Scale GPU pool linearly; prompt caching becomes critical to avoid 10x compute growth.
- **Session Managers**: Stateless horizontally scaled; session state moves to Redis.
- **Gateway**: Add regional PoPs to reduce WebSocket latency.
- **Prompt cache hit rate** becomes the key efficiency lever -- invest in cache-aware routing.

### At 100x (350K QPS, 100M users)
- **Multi-region inference**: Deploy model replicas in 5+ regions; route by latency.
- **Tiered storage**: Hot conversations in Redis, warm in PostgreSQL, cold in S3.
- **Model distillation**: Train smaller specialized models for common tasks (code completion, test generation) to reduce GPU cost.
- **Speculative decoding**: Use a draft model to predict tokens, verify with the full model -- 2-3x throughput improvement.
- **Disaggregated inference**: Separate prefill (compute-bound) from decode (memory-bound) onto different GPU types.

---

## 11) Reliability and Fault Tolerance (3-5 min)

- **GPU failure**: Inference Router detects unhealthy nodes via health checks (every 5s); reroutes requests; KV-cache is lost but rebuilt on next turn.
- **Session Manager crash**: Session state persisted in Redis; client reconnects and resumes from last completed turn.
- **Client disconnect**: Server keeps session alive for 30 minutes; client can reconnect and continue.
- **Model overload**: Circuit breaker returns 429 with retry-after; client shows queue position or falls back to smaller model.
- **Cascading failure prevention**: Each component has independent circuit breakers; inference is isolated per model tier.
- **Data durability**: Conversation logs written to durable storage asynchronously; tool execution results are ephemeral (reconstructable).

---

## 12) Observability and Operations (2-4 min)

**Key Metrics:**
- Time-to-first-token (TTFT) p50/p95/p99
- Inter-token latency (ITL) p50/p95/p99
- Tool execution latency by tool type
- Agentic loop iterations per task (measures efficiency)
- Prompt cache hit rate
- GPU utilization and KV-cache occupancy
- Tokens per second per GPU (throughput)
- Error rate by category (inference timeout, tool failure, context overflow)

**Alerting:**
- TTFT p99 > 2s for > 5 minutes
- Cache hit rate drops below 50%
- GPU utilization > 90% sustained (capacity planning trigger)
- Tool error rate > 5% (potential client version issue)

**Dashboards:**
- Real-time inference throughput and latency heatmap
- Per-model queue depth and routing decisions
- Conversation length distribution (context growth patterns)
- Cost per request by model tier

---

## 13) Security (1-3 min)

- **Authentication**: API keys or OAuth tokens validated at the Gateway; keys scoped to org/user.
- **Tool sandboxing**: Client-side tools run in the user's environment with their permissions; the server never executes arbitrary code.
- **Permission gates**: Destructive operations (file delete, git push) require explicit user approval regardless of permission mode.
- **Prompt injection defense**: Tool results are clearly delimited in the context; the model is instructed to treat tool output as data, not instructions.
- **Data privacy**: Conversation content is encrypted in transit (TLS 1.3) and at rest (AES-256); opt-out of logging for enterprise users.
- **Abuse prevention**: Rate limiting per user/org; content filtering on outputs; token budget caps.
- **SSRF prevention**: Server-side web fetches validate URLs against allowlists; block private IP ranges.

---

## 14) Team and Operational Considerations (1-2 min)

**Team structure (for a team of ~30):**
- **Client team** (5): CLI, IDE extensions, permission system, local tool execution.
- **Inference platform team** (8): GPU cluster, continuous batching, KV-cache, model loading.
- **Session/orchestration team** (6): Agentic loop, context management, prompt caching, streaming.
- **API/gateway team** (4): Auth, rate limiting, WebSocket infrastructure.
- **Safety/trust team** (3): Content filtering, permission model, abuse detection.
- **Infrastructure/SRE team** (4): Deployment, monitoring, cost optimization.

**Deployment:**
- Client: versioned releases via package managers (npm, brew, pip); auto-update with opt-out.
- Backend: canary deployments per region; model updates are blue-green with traffic shifting.
- GPU firmware/driver updates: rolling restarts with drain-before-restart.

**On-call rotation:**
- Inference on-call: GPU health, model serving, latency spikes.
- Platform on-call: Gateway, session management, billing.

---

## 15) Common Follow-up Questions

### "How do you handle a conversation that exceeds the context window?"
Context compression: summarize older turns, keep recent turns verbatim. The compressor preserves file paths, variable names, and decisions. Users can also start a new conversation with a /compact command that carries forward a summary.

### "How do you prevent the LLM from running dangerous commands?"
Three layers: (1) the LLM is instructed to confirm before destructive actions, (2) the client enforces a permission model with allow/deny lists per tool category, (3) specific patterns (rm -rf /, DROP TABLE) trigger mandatory confirmation regardless of settings.

### "Why not run tools server-side for lower latency?"
File and shell operations require access to the user's local environment -- their codebase, their git state, their running processes. Uploading the entire workspace adds latency and raises privacy concerns. Server-side execution is used only for stateless operations (web search, documentation lookup).

### "How do you handle multiple concurrent tool calls?"
The LLM can emit multiple tool_use blocks. The client checks for dependencies (explicit or inferred), executes independent tools in parallel, and returns all results together. This reduces round-trips from N sequential inference calls to 1.

### "How do you optimize cost for simple queries?"
Multi-model routing: a lightweight classifier (or user toggle) routes simple queries to a smaller, cheaper model. Prompt caching reduces redundant computation. Batching amortizes GPU overhead across concurrent users.

---

## 16) Closing Summary (30-60s)

> "The system is a WebSocket-connected agentic loop. The CLI client connects to an API Gateway, which routes to a Session Manager that orchestrates multi-turn conversations. The Session Manager assembles prompts -- leveraging prefix caching for 70-85% cache hit rates -- and sends them to an Inference Router that places requests on the optimal GPU pool. The Inference Engine streams tokens back via continuous batching. When the LLM emits tool calls, the Session Manager pauses streaming and delegates execution to the client, which runs tools locally with permission gates. Results feed back into the context for the next inference round. Context compression keeps conversations within window limits. The system scales horizontally at every layer: stateless gateways, Redis-backed session state, cache-aware inference routing, and multi-region GPU pools. Safety is enforced through client-side sandboxing, permission modes, and content filtering."

---

## Appendix A: Configuration / Tuning Knobs

| Parameter | Default | Notes |
|-----------|---------|-------|
| `max_context_tokens` | 200,000 | Model-dependent |
| `compression_threshold` | 0.8 | Trigger compression at 80% of window |
| `ttft_target_ms` | 500 | Alert if exceeded at p99 |
| `itl_target_ms` | 50 | Inter-token latency target |
| `tool_timeout_ms` | 120,000 | 2 min for shell commands |
| `session_idle_ttl_min` | 30 | Keep session alive after disconnect |
| `cache_eviction_idle_s` | 300 | Drop KV-cache after 5 min idle |
| `max_tool_output_lines` | 2,000 | Truncate large tool results |
| `max_agentic_iterations` | 50 | Safety limit on loop iterations |

---

## Appendix B: Vocabulary Cheat Sheet

- **Agentic loop**: Multi-step cycle where the LLM reasons, calls tools, processes results, and continues until the task is done.
- **TTFT (Time-to-First-Token)**: Latency from request submission to the first streamed token.
- **ITL (Inter-Token Latency)**: Time between consecutive streamed tokens.
- **Prompt/prefix caching**: Reusing KV-cache computations for repeated prompt prefixes across turns.
- **Continuous batching**: Dynamically adding/removing requests from a running GPU batch to maximize throughput.
- **Context compression**: Summarizing older conversation turns to free token budget.
- **Tool sandboxing**: Restricting tool execution to safe operations with user approval gates.
- **Speculative decoding**: Using a small draft model to predict tokens, verified by the full model for faster generation.

---

## Appendix C: References

- [41_llm_inference_system](../41_llm_inference_system/) -- GPU inference architecture, continuous batching, KV-cache
- [42_model_serving_autoscaling](../42_model_serving_autoscaling/) -- Autoscaling GPU pools
- [51_multi_model_routing](../51_multi_model_routing/) -- Cost-aware model selection
- [52_context_window_memory_system](../52_context_window_memory_system/) -- Context management strategies
- [17_websocket_notification_system](../17_websocket_notification_system/) -- WebSocket architecture patterns
- [26_rate_limiter](../26_rate_limiter/) -- Rate limiting strategies
- [49_safety_moderation_system](../49_safety_moderation_system/) -- Content safety and moderation
