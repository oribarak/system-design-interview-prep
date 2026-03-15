# System Design Interview: Prompt Engineering Playground -- "Perfect Answer" Playbook

## 0) Opening: Frame the Problem (30-60s)

> "We need to design a prompt engineering playground — think Anthropic Console or OpenAI Playground — where developers can interactively craft, test, and iterate on prompts against LLM APIs. This is a product-focused system design from a client engineer's perspective. The core challenge is building a responsive, real-time UI that streams token-by-token responses, supports complex prompt configurations (system prompts, multi-turn conversations, tool definitions, parameter tuning), maintains version history for prompt iterations, and enables side-by-side comparison of outputs across models and parameters. I'll focus on the client architecture, state management, streaming UX, and the backend APIs that power it."

## 1) Clarifying Questions (2-5 min)

| # | Question | Typical Answer |
|---|----------|---------------|
| 1 | Is this a web application or does it need native desktop/mobile? | Web application, primarily desktop-focused. |
| 2 | How many concurrent users do we expect? | ~50K DAU, ~5K concurrent at peak. |
| 3 | Do we need to support multiple LLM providers or just our own? | Primarily our own API, but pluggable for future providers. |
| 4 | Do we need collaboration features (shared prompts, team workspaces)? | Yes, basic sharing and team workspaces. |
| 5 | Do we need prompt versioning and diff? | Yes, git-like version history per prompt. |
| 6 | What's the max context length we need to support in the editor? | Up to 200K tokens (full context window). |
| 7 | Do we need to support tool/function calling in the playground? | Yes, users define tool schemas and see tool call results. |
| 8 | Do we need A/B comparison — running the same prompt against two models side-by-side? | Yes, critical feature. |

## 2) Back-of-Envelope Estimation (3-5 min)

**Traffic:**
- 50K DAU, average 20 prompt runs per session = 1M prompt executions/day.
- Peak: ~30 QPS prompt executions (3x average).
- Average input: 1,500 tokens, average output: 500 tokens.
- Side-by-side comparisons: ~20% of runs = 200K dual-model runs/day.

**Storage:**
- Prompt versions: 50K users x 50 prompts x 10 versions = 25M versions.
- Average prompt config: ~5 KB (system prompt, messages, tool defs, params) = 125 GB total.
- Conversation logs (inputs + outputs): 1M/day x 10 KB avg = 10 GB/day, ~3.6 TB/year.

**Bandwidth:**
- Streaming: 30 QPS x 500 tokens x 5 bytes/token x 8 bytes SSE overhead = ~1.2 MB/s outbound (modest).
- Side-by-side doubles the streaming bandwidth for those requests.

**Latency:**
- TTFT target: < 1s (user is watching the output area, expects near-instant feedback).
- Perceived responsiveness: editor operations (save, load, version switch) < 200ms.

## 3) Requirements

### Functional
- FR1: Compose prompts with system message, user/assistant message turns, and parameter controls (temperature, top-p, max tokens, stop sequences).
- FR2: Stream LLM responses token-by-token in real time with syntax highlighting.
- FR3: Support tool/function call definitions, execution simulation, and result display.
- FR4: Side-by-side comparison mode — same prompt, different models or parameters.
- FR5: Version history with diff view for each prompt project.
- FR6: Team workspaces with shared prompt libraries and access control.
- FR7: Export prompts as API code snippets (Python, TypeScript, cURL).

### Non-Functional
- NFR1: TTFT < 1s for interactive prompt runs.
- NFR2: Editor responsiveness < 200ms for all local operations.
- NFR3: Support 200K token context windows without UI freezing.
- NFR4: 99.9% availability for the playground service.
- NFR5: Prompt data encrypted at rest, tenant-isolated.

## 4) API Design

```
// Execute a prompt — tied to the prompt resource, not a standalone endpoint.
// This ensures every execution is automatically linked to a prompt for history tracking.
POST /v1/prompts/{prompt_id}/execute
Headers:
  Authorization: Bearer <api_key>
  Idempotency-Key: <client-generated-uuid>   // Prevents duplicate expensive model calls on retry
Body:
{
  "model": "claude-sonnet-4-6",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant..." },
    { "role": "user", "content": "Explain quantum computing" }
  ],
  "tools": [
    { "name": "get_weather", "description": "...", "input_schema": { ... } }
  ],
  "parameters": {
    "temperature": 0.7,
    "max_tokens": 1024,
    "top_p": 0.9,
    "stop_sequences": ["\n\nHuman:"]
  },
  "stream": true
}
Response (streaming): Server-Sent Events
  data: {"type": "content_block_delta", "delta": {"text": "Quantum"}}
  data: {"type": "content_block_delta", "delta": {"text": " computing"}}
  data: {"type": "tool_use", "name": "get_weather", "input": {"city": "SF"}}
  ...
  data: {"type": "message_stop", "usage": {"input_tokens": 45, "output_tokens": 312}}

// Why POST /prompts/:id/execute instead of POST /execute?
// - Execution is inherently tied to a prompt — automatic history tracking
// - No orphaned executions; every run is linked to a prompt and version
// - Clean REST semantics: execution is a sub-resource of a prompt
// - Client-first thinking: the execution history screen shows all runs for a prompt

POST /v1/prompts/{prompt_id}/compare
Body:
{
  "configs": [
    { "model": "claude-sonnet-4-6", "parameters": { "temperature": 0.3 } },
    { "model": "claude-opus-4-6", "parameters": { "temperature": 0.7 } }
  ],
  "messages": [ ... ]
}
Headers:
  Idempotency-Key: <client-generated-uuid>
Response: Two parallel SSE streams multiplexed with stream_id tags.

POST /v1/prompts
Body: { "name": "My Prompt", "workspace_id": "w-123", "config": { ... } }
Response: { "prompt_id": "p-abc123", "version": 1 }

// Autosave with PATCH — for constant prompt tweaking without creating new versions
PATCH /v1/prompts/{prompt_id}
Body: { "config": { "parameters": { "temperature": 0.5 } } }
Response: { "prompt_id": "p-abc123", "updated_at": "..." }
// PATCH updates the draft state without creating a version.
// Versions are created explicitly (Cmd+S) or automatically on execute.

PUT /v1/prompts/{prompt_id}
Body: { "config": { ... }, "commit_message": "Adjusted temperature" }
Response: { "prompt_id": "p-abc123", "version": 2 }

// Execution history — cursor-based pagination for potentially long history
GET /v1/prompts/{prompt_id}/executions?cursor=<opaque_cursor>&limit=20
Response: {
  "executions": [
    { "execution_id": "e-xyz", "version": 2, "model": "claude-sonnet-4-6",
      "input_tokens": 45, "output_tokens": 312, "latency_ms": 1200, "created_at": "..." }
  ],
  "next_cursor": "eyJ0IjoiMjAyNi...",
  "has_more": true
}
// Cursor-based (not offset) because: stable under concurrent inserts,
// no skipping/duplicating results when new executions are added during pagination.

GET /v1/prompts/{prompt_id}/versions
Response: { "versions": [{ "version": 2, "diff": "...", "created_at": "..." }, ...] }

GET /v1/prompts/{prompt_id}/versions/{version}
Response: { "config": { ... }, "version": 2 }

// Share a prompt (creates a read-only snapshot)
POST /v1/prompts/{prompt_id}/share
Body: { "visibility": "workspace" | "public", "expires_at": "..." }
Response: { "shared_prompt_id": "sp-abc", "share_url": "..." }
```

## 5) Data Model

### Entities

| Entity | Key Fields | Storage |
|--------|-----------|---------|
| User | user_id, email, api_key_hash, created_at | PostgreSQL |
| Organization | org_id, name, owner_id, plan_tier, token_budget | PostgreSQL |
| Project | project_id, org_id, name, description, created_at | PostgreSQL |
| Prompt | prompt_id, project_id, name, current_version, draft_config (JSONB), created_at | PostgreSQL |
| PromptVersion | version_id, prompt_id, version_num, config (JSONB), commit_message, created_by, created_at | PostgreSQL |
| Execution | execution_id, prompt_id, version_id, model, input_tokens, output_tokens, latency_ms, idempotency_key, created_at | PostgreSQL (metadata) + ClickHouse (analytics) |
| ExecutionResult | execution_id, full_output, tool_calls[], finish_reason, raw_response | S3 (large outputs) + PostgreSQL (metadata) |
| SharedPrompt | shared_prompt_id, prompt_id, version_id, visibility, share_url, expires_at, created_by | PostgreSQL |

**Key entity relationships (client-first thinking -- what data does each screen need?):**
- **Prompt list screen**: `GET /projects/:id/prompts` -- needs Prompt.name, current_version, last execution stats
- **Prompt editor screen**: `GET /prompts/:id` -- needs Prompt.draft_config (JSONB), version history sidebar
- **Execution history screen**: `GET /prompts/:id/executions` -- cursor-paginated list of Execution + ExecutionResult summaries
- **Shared prompt view**: `GET /shared/:shared_prompt_id` -- needs SharedPrompt -> PromptVersion.config (read-only)

### Storage Choices
- **Prompt configs**: PostgreSQL with JSONB — supports complex nested structures, indexable, transactional versioning.
- **Run outputs**: S3 for full conversation logs (can be large with 200K context), metadata in PostgreSQL.
- **Analytics**: ClickHouse for run metrics (token usage, latency distributions, model comparison).
- **Session state**: Redis for active editor sessions. Auto-save uses `PATCH /prompts/:id` to persist draft_config to PostgreSQL every 5 seconds (debounced). Redis caches the latest draft for fast loads.

## 6) High-Level Architecture (5-8 min)

**One-sentence summary:** A React-based SPA with a split-pane editor communicates over SSE with a backend that proxies prompt executions to the inference API, while a versioning service provides git-like prompt history and a comparison engine runs parallel model evaluations.

### Components

1. **Frontend SPA** — React app with split-pane layout: prompt editor (left), output viewer (right). Monaco editor for prompt composition with syntax highlighting.
2. **API Gateway** — Authentication, rate limiting, request routing.
3. **Playground API Service** — Handles prompt CRUD, version management, run orchestration.
4. **Streaming Proxy** — Proxies SSE connections to the inference API, adds metadata (token counting, latency tracking), handles comparison multiplexing.
5. **Inference API** — The actual LLM serving layer (existing infrastructure).
6. **Version Store** — Manages prompt version history, diffs, branching.
7. **Comparison Engine** — Orchestrates parallel model runs, synchronizes and merges streams.
8. **Analytics Service** — Tracks run metrics, token usage, cost estimation.
9. **Collaboration Service** — Real-time cursors, workspace sharing, access control.

### Data Flow
1. User edits prompt in Monaco editor. Changes auto-saved to Redis every 5 seconds.
2. User clicks "Run". Frontend opens SSE connection to Streaming Proxy.
3. Streaming Proxy forwards the prompt to Inference API, begins streaming tokens back.
4. Frontend renders tokens incrementally in the output pane with syntax highlighting.
5. If a tool_use block arrives, the UI renders the tool call and lets the user provide a simulated result (or auto-execute if configured).
6. On completion, the run metadata (tokens, latency, cost) is logged to ClickHouse.
7. User can save the prompt, creating a new version in the Version Store.

## 7) Deep Dive #1: Client-Side Streaming Architecture (10-15 min)

### The challenge: rendering a high-frequency token stream without janking the UI

LLM streaming sends tokens every 20-50ms. At 50 tokens/second, the UI must:
1. Parse the SSE event.
2. Update the state with the new token.
3. Re-render the output pane (potentially with syntax highlighting, markdown rendering).
4. Auto-scroll to the latest token.
5. Do all of this without dropping frames (16ms budget at 60fps).

Naive approach: `setState` on every token. This triggers a React re-render 50 times/second, each time re-rendering the entire output string. With a 5,000-token output, that's re-processing 5,000 tokens of markdown/syntax highlighting 50 times per second. The UI freezes.

### Solution: buffered rendering with requestAnimationFrame

```
Token arrives via SSE
  -> Append to a mutable buffer (no React state update)
  -> On next requestAnimationFrame (16ms tick):
       -> Flush buffer: take all accumulated tokens since last frame
       -> Append to a DOM text node directly (bypass React for the hot path)
       -> Update React state only for metadata (token count, cost estimate)
       -> Trigger auto-scroll
```

**Why this works:**
- Tokens arrive at ~50/s but frames render at 60fps. Multiple tokens batch into a single frame.
- Direct DOM manipulation for the text content avoids React's reconciliation overhead.
- React state only updates for metadata displayed in a separate component (non-blocking).

### Handling large outputs (200K tokens)

When output exceeds ~10K tokens, rendering the entire text becomes expensive even once per frame.

**Virtualized output rendering:**
- Only render the visible portion of the output (like react-virtualized for text).
- Keep the full text in a plain string buffer.
- The visible viewport shows only ~100 lines. As the user scrolls, render new lines on demand.
- During streaming, pin to bottom — only append new text to the visible tail.

### Syntax highlighting during streaming

Highlighting the entire output on every token is O(n) per token = O(n^2) total. For code blocks in the LLM response:
- Use an incremental parser (Tree-sitter/Lezer) that can highlight just the new tokens.
- Detect code block boundaries (``` markers) and only apply highlighting within them.
- Highlight lazily: apply full highlighting only after the code block closes or streaming ends.

### Side-by-side comparison streaming

Two SSE streams arrive at different rates (different models, different speeds). The UI must:
1. Open two SSE connections simultaneously (or one multiplexed connection with stream_id tags).
2. Render each stream independently in its own pane.
3. Show a "waiting" indicator on the faster stream's pane after it finishes, until the slower one catches up.
4. On both completion, show comparison metrics: token count diff, latency diff, cost diff.

### Connection management

- **Reconnection**: If SSE drops mid-stream, attempt reconnection with `Last-Event-ID` header to resume from where we left off. The server buffers the last 30 seconds of events.
- **Cancellation**: User clicks "Stop". Frontend closes the SSE connection. Backend detects the closed connection and cancels the inference request (critical — frees GPU resources).
- **Timeout**: If no token arrives for 30s, show a timeout warning. After 60s, auto-cancel.

## 8) Deep Dive #2: Prompt Versioning System (8-12 min)

### Why this matters
Prompt engineering is iterative. A developer might tweak a system prompt 50 times in an hour. They need to go back to version 37 because version 38-50 made things worse. Without versioning, this is "undo" with limited depth.

### Design: content-addressable storage (git-inspired)

Each prompt version is stored as an immutable snapshot:

```
PromptVersion:
  version_id: UUID
  prompt_id: FK -> Prompt
  parent_version_id: UUID (null for first version)
  config_hash: SHA-256 of the serialized config
  config: JSONB {
    system_prompt: "...",
    messages: [...],
    tools: [...],
    parameters: { temperature, max_tokens, ... },
    model: "claude-sonnet-4-6"
  }
  commit_message: "Lowered temperature to reduce hallucination"
  created_by: user_id
  created_at: timestamp
```

### Auto-save vs. explicit save

- **Auto-save**: Every 5 seconds, save the current editor state to Redis as a draft. This is NOT a version — it's crash recovery.
- **Explicit save**: User clicks "Save Version" or presses Cmd+S. This creates a new `PromptVersion` row. The user can optionally add a commit message.
- **Auto-version on run**: Every time the user clicks "Run", if the config has changed since the last version, auto-create a new version. This ensures every run is reproducible.

### Diff computation

Prompt configs are JSON. Compute diffs using JSON-patch (RFC 6902):

```json
[
  { "op": "replace", "path": "/parameters/temperature", "value": 0.3 },
  { "op": "add", "path": "/messages/2", "value": { "role": "user", "content": "..." } }
]
```

The UI renders diffs as:
- **Parameter changes**: highlighted key-value pairs (red/green).
- **System prompt changes**: inline text diff (like GitHub).
- **Message list changes**: added/removed/modified messages with visual markers.

### Version timeline UI

Display a vertical timeline on the side panel:
- Each node = a version, showing commit message, timestamp, who made it.
- Click any version to load it into the editor.
- "Compare" button opens a split diff view between any two versions.
- "Fork" from any historical version to create a new branch of exploration.

### Branching (advanced)

For complex prompt engineering, support branching:
- "Fork" from version 12 to try a different approach.
- Each branch has its own version history.
- "Merge" isn't needed (prompts don't have merge conflicts in the git sense) — but users can copy specific fields from one branch to another.

### Storage optimization

Many versions differ by only a few fields (e.g., changing temperature from 0.7 to 0.3). But storing full snapshots is simpler and cheap:
- At 5 KB per version, 25M versions = 125 GB. PostgreSQL handles this easily.
- For the rare 200K-token system prompt, store the text in S3 and reference it by hash. Deduplicate identical content via content-addressing.

## 9) Trade-offs and Alternatives (3-5 min)

| Decision | Option A | Option B | Our Choice |
|----------|----------|----------|-----------|
| Rendering strategy | React state per token | requestAnimationFrame buffered DOM | Buffered DOM — 60fps instead of janky |
| Editor component | Textarea | Monaco Editor | Monaco — syntax highlighting, autocomplete, large text support |
| Streaming protocol | SSE | WebSocket | SSE — simpler, unidirectional is sufficient, HTTP/2 multiplexing for comparison |
| Version storage | Full snapshots | Delta/patch chain | Full snapshots — simpler reads, 5 KB/version is cheap |
| Output rendering | Full re-render | Virtualized/incremental | Virtualized for large outputs, full render for small ones |
| Collaboration | Polling | CRDT/OT real-time | Polling for v1, CRDT for v2 (complexity vs. value) |

## 10) Scaling: 10x and 100x (3-5 min)

### 10x (500K DAU, 300 QPS prompt runs)
- Streaming Proxy scales horizontally — stateless except for the SSE connection.
- Add Redis Cluster for session state and draft auto-save.
- CDN for static assets (the SPA itself, Monaco editor resources).
- Read replicas on PostgreSQL for prompt version reads.
- ClickHouse handles analytics at this scale natively.

### 100x (5M DAU, 3K QPS prompt runs)
- Shard prompt storage by workspace_id — each workspace's prompts on a dedicated shard.
- Move to a dedicated streaming service with connection pooling to the inference API (avoid opening 3K simultaneous inference connections).
- Prompt template marketplace — popular prompts cached at the CDN edge.
- Regional deployment: streaming proxies in each region to minimize TTFT latency.
- Rate limiting per workspace to prevent abuse of expensive model runs.

## 11) Reliability and Fault Tolerance (3-5 min)

- **SSE disconnect**: Client auto-reconnects with `Last-Event-ID`. Server replays buffered events (30s window). If gap is too large, return a "reconnect_failed" event and the client shows partial output with a "resume" button.
- **Draft loss prevention**: Auto-save to Redis every 5s. On browser crash/close, the draft is recoverable. Redis is replicated (Sentinel/Cluster).
- **Version store failure**: PostgreSQL with synchronous replication. Losing a version is unacceptable — users rely on version history.
- **Inference API timeout**: If no token in 30s, surface a clear error. Don't leave the user staring at a spinner. Offer "retry" and "try a different model" buttons.
- **Rate limiting**: Per-user and per-workspace token budgets. Show remaining quota in the UI. Return 429 with `Retry-After` and a clear message about which limit was hit.
- **Graceful degradation**: If comparison mode is overloaded, fall back to sequential runs instead of parallel. If analytics is down, still serve the core playground experience.

## 12) Observability and Operations (2-4 min)

### Key Metrics
- **TTFT p50/p99** — per model, per region.
- **Stream completion rate** — % of streams that complete vs. error/cancel.
- **Editor latency** — time from keystroke to render (client-side RUM).
- **Version save latency** — time to persist a new version.
- **Comparison drift** — time difference between the two streams completing.
- **Token usage** — per user, per workspace, per model.

### Dashboards
- Real-time streaming health: active connections, error rate, reconnection rate.
- Token usage and cost dashboard per workspace (for billing visibility).
- Prompt versioning activity: versions created/day, average versions per prompt.

### Alerting
- Stream error rate > 5% for any model.
- TTFT p99 > 3s.
- Redis draft store latency > 100ms.
- Version save failures > 0.1%.

## 13) Security (1-3 min)

- **Authentication**: OAuth2/SSO for enterprise workspaces. API keys for programmatic access.
- **Authorization**: RBAC per workspace — owner, editor, viewer roles. Editors can run prompts and save versions. Viewers can only see outputs.
- **Prompt data isolation**: Workspace data is tenant-isolated in the database. No cross-tenant prompt leakage.
- **API key handling**: API keys stored encrypted. Never logged in plaintext. Masked in the UI after creation.
- **Content security**: CSP headers to prevent XSS (especially important since we render LLM output that could contain scripts).
- **Output sanitization**: LLM output rendered in a sandboxed iframe or with strict HTML sanitization to prevent injection.
- **Audit log**: All prompt runs, version changes, and sharing actions logged for compliance.

## 14) Team and Operational Considerations (1-2 min)

- **Frontend Team** (3-4 engineers): Editor UX, streaming renderer, comparison UI, version timeline.
- **Backend/API Team** (2-3 engineers): Playground API, version store, streaming proxy.
- **Platform Team** (2 engineers): Integration with inference API, analytics pipeline, rate limiting.
- **Design** (1-2 designers): Editor UX, output rendering, diff visualization.
- **On-call**: Frontend on-call for client-side error spikes (Sentry). Backend on-call for streaming proxy and version store.

## 15) How Token-by-Token Streaming UX Actually Works

The user clicks "Run" and sees tokens appear one at a time. Here's the full path:

1. **Click "Run"**: Frontend validates the prompt config, opens an SSE connection to `/v1/playground/run?stream=true`.
2. **Backend receives**: Streaming Proxy authenticates, logs the run, forwards the request to the Inference API with `stream: true`.
3. **First token arrives**: Inference API sends the first SSE event. Proxy adds metadata (timestamp, event ID) and forwards to the client. **This is TTFT** — the user sees the first character appear.
4. **Buffered rendering**: Tokens accumulate in a mutable buffer. On the next animation frame (~16ms), all buffered tokens are flushed to the DOM. A cursor blinks at the insertion point.
5. **Tool call handling**: If the stream emits a `tool_use` event, the output pane renders a tool call card showing the function name, arguments (syntax-highlighted JSON), and a "Provide Result" input. The user types a mock result, clicks "Submit", and the result is sent back to continue generation.
6. **Completion**: `message_stop` event arrives. The UI shows final metrics: total tokens, latency, estimated cost. The "Run" button re-enables. The output is saved to the run history.
7. **Cancel**: User clicks "Stop" mid-stream. Frontend closes the SSE connection. Backend detects the close and cancels the inference request, freeing GPU resources.

**Key UX details:**
- During streaming, the "Run" button becomes a "Stop" button (red).
- A progress indicator shows tokens generated so far and estimated cost accruing in real time.
- The output pane auto-scrolls during streaming but stops if the user manually scrolls up (reading earlier output). A "Jump to bottom" button appears.
- Markdown rendering is applied lazily — during streaming, show raw text. On completion, render markdown with syntax-highlighted code blocks.

## 16) Common Follow-up Questions

1. **How do you handle very long system prompts (50K+ tokens) in the editor?**
   - Monaco editor handles large documents well (it powers VS Code). For the prompt config, store the system prompt as a separate S3-backed blob referenced by hash. The editor loads it lazily and uses virtual scrolling.

2. **How do you implement the code snippet export feature?**
   - Template-based code generation. Store templates for Python (anthropic SDK), TypeScript, cURL. Inject the prompt config (messages, parameters, tool definitions) into the template. Handle escaping and formatting per language. Show a copy-to-clipboard button with syntax highlighting.

3. **How do you handle tool/function calling in the playground?**
   - When the LLM emits a `tool_use` block, pause the stream and render a tool call UI card. Three options: (1) Manual — user types a mock result. (2) Auto — execute against a user-defined mock server URL. (3) Simulated — use a small model to generate a plausible tool result. The result is appended to messages and generation continues.

4. **How do you prevent users from running expensive prompts accidentally?**
   - Show a cost estimate before running (based on input tokens x model pricing). For runs exceeding a threshold ($5), show a confirmation dialog. Enforce per-workspace daily budgets. Show a running cost counter during streaming.

5. **How do you handle prompt sharing and discovery?**
   - Users can publish prompts to a workspace library (private) or a public gallery. Published prompts are read-only snapshots. Other users can "fork" them into their own workspace. Attribution and usage stats tracked.

6. **How would you add evaluation/scoring to the playground?**
   - After a run completes, let users score the output (thumbs up/down, 1-5 stars, custom rubric). Store scores linked to the run and prompt version. Aggregate scores per version to track which prompt iterations improve quality. This becomes the foundation for systematic prompt optimization.

## 17) Closing Summary (30-60s)

> "The prompt engineering playground is a product-focused system design centered on real-time streaming UX and iterative workflow. The key technical challenges are rendering a high-frequency token stream at 60fps without UI jank — solved with requestAnimationFrame buffering and direct DOM manipulation — and providing git-like version history for prompt iterations so developers can track what changed and roll back when needed. Side-by-side comparison runs two inference streams in parallel with synchronized UI rendering. The architecture is a React SPA with Monaco editor, backed by a streaming proxy that handles SSE multiplexing, a PostgreSQL-based version store with content-addressable snapshots, and ClickHouse for run analytics. It scales horizontally through stateless streaming proxies and CDN-cached static assets."

## Appendix A: Configuration / Tuning Knobs

| Knob | Default | Tuning Rationale |
|------|---------|-----------------|
| auto_save_interval | 5s | Shorter = less draft loss risk, higher Redis write load |
| stream_buffer_flush_interval | 16ms (1 frame) | Matches 60fps. Lower = smoother but more DOM writes |
| max_output_render_tokens | 10K | Beyond this, switch to virtualized rendering |
| sse_reconnect_buffer | 30s | Longer = more memory per connection on the proxy |
| version_auto_create_on_run | true | false = fewer versions but runs become non-reproducible |
| comparison_max_parallel | 3 | More = higher GPU load per user |
| draft_ttl | 24h | How long unsaved drafts persist in Redis |
| cost_confirmation_threshold | $5 | Prompts estimated above this show a confirmation dialog |

## Appendix B: Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| SSE | Server-Sent Events — unidirectional streaming protocol over HTTP |
| Monaco | The code editor that powers VS Code, used for prompt composition |
| TTFT | Time to first token — latency from clicking "Run" to seeing the first character |
| Content-addressable | Storage where the key is a hash of the content (deduplicates identical versions) |
| JSON Patch (RFC 6902) | Standard format for describing changes between two JSON documents |
| requestAnimationFrame | Browser API to schedule work on the next repaint (60fps) |
| Virtual scrolling | Rendering only visible rows/lines, recycling DOM nodes as user scrolls |
| CRDT | Conflict-free Replicated Data Type — for real-time collaborative editing |
| Prompt version | An immutable snapshot of the full prompt configuration at a point in time |
| Idempotency-Key | Client-generated UUID sent as a header to prevent duplicate model executions on retry |
| Cursor-based pagination | Using an opaque cursor token (not offset) for stable pagination under concurrent writes |
| Execution (Run) | A single execution of a prompt against a model, producing an output |

## Appendix C: References

- Monaco Editor (Microsoft, powers VS Code)
- Server-Sent Events (W3C specification)
- JSON Patch (RFC 6902)
- React Virtualized / react-window for large list rendering
- Tree-sitter / Lezer for incremental syntax highlighting
- Anthropic Console and OpenAI Playground as product references
