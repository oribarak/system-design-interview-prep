# Real-Time AI Coding Assistant (Claude Code) -- Simplified Interview Version

This is the whiteboard-friendly version. Use this in a 45-minute interview where the interviewer wants breadth over implementation depth.

---

## 1. Problem Statement (30s)
We need a real-time AI coding assistant that accepts natural language instructions, streams LLM responses token-by-token, and autonomously executes tools (file reads, edits, shell commands) in a multi-turn agentic loop. The hard part is orchestrating the tool execution loop with low latency, managing context windows that grow across hundreds of turns, and keeping prompt caching efficient so that repeated prefixes don't re-compute attention.

## 2. Requirements
**Functional:** Stream LLM responses token-by-token. Execute tool calls (file read/write, shell, web search) within the conversation loop. Support multi-turn conversations with growing context. User-controlled permission gates for tool calls. Model selection (fast vs full mode). Automatic context compression when approaching window limits.

**Non-functional:** Time-to-first-token under 500ms with prompt caching. Inter-token latency under 50ms. 99.9% availability. Sandboxed tool execution with permission gates. Handle 3,500+ concurrent sessions at peak. 5-10x cost savings via multi-model routing.

## 3. Core Concept: The Agentic Tool Execution Loop
The LLM generates text and tool_use blocks. When a tool_use block is emitted, the Session Manager pauses streaming and sends the tool request to the client. The client executes the tool locally (file read, shell command) with permission checks, returns the result, and the Session Manager appends it to context and re-invokes inference. This loop repeats until the LLM produces a final response with no tool calls. Tools run client-side because the user's codebase lives locally -- no need to upload it. Independent tools execute in parallel; dependent tools run sequentially.

## 4. High-Level Architecture
```
CLI Client <-- WebSocket --> [API Gateway] --> [Session Manager]
                                                    |
                                             [Inference Router]
                                                    |
                                             [GPU Inference Engine]
                                             (continuous batching + KV-cache)
                                                    |
                                             [Prompt Cache] -- prefix KV-cache reuse
                                                    |
                                             [Context Compressor] -- summarize old turns
```
- **Session Manager**: Orchestrates the agentic loop. Assembles prompts, dispatches tool requests to client, manages context budget.
- **Inference Router**: Routes to optimal GPU pool based on model, load, and prompt cache locality.
- **GPU Inference Engine**: Continuous batching with KV-cache management. Streams tokens back through the chain.
- **Prompt Cache**: Reuses KV-cache for repeated prompt prefixes (system prompt + prior turns). 70-85% hit rate, reducing TTFT from 2s to 300ms.
- **Context Compressor**: Summarizes older turns when context exceeds 80% of the window.

## 5. How a Coding Task Gets Completed
1. User types a natural language instruction in the CLI.
2. CLI sends the message over WebSocket to the Session Manager.
3. Session Manager assembles the full prompt: system prompt + conversation history + current message.
4. Inference Router checks for prompt cache hit and routes to the optimal GPU.
5. Inference Engine streams tokens back; CLI renders them incrementally.
6. When the LLM emits a tool_use block (e.g., "read file X"), streaming pauses.
7. Client executes the tool locally (with permission check), returns the result over WebSocket.
8. Session Manager appends the tool result to context and re-invokes inference.
9. Loop continues until the LLM produces a final text response with no tool calls.

## 6. What Happens When Things Fail?
- **GPU failure**: Inference Router reroutes to healthy nodes. KV-cache is lost but rebuilt on next turn.
- **Client disconnect**: Session state persisted in Redis. Client can reconnect and resume within 30 minutes.
- **Model overload**: Circuit breaker returns 429 with retry-after. Client can fall back to a smaller, faster model.

## 7. Scaling
- **10x**: Prompt caching becomes the key efficiency lever -- invest in cache-aware routing to avoid 10x compute growth. Stateless Session Managers with Redis-backed state. Regional gateway PoPs.
- **100x**: Multi-region inference with model replicas in 5+ regions. Speculative decoding (draft model predicts, full model verifies) for 2-3x throughput. Disaggregated inference: separate prefill (compute-bound) and decode (memory-bound) onto different GPU types.

## 8. Key Trade-offs to Mention
| Decision | Trade-off |
|----------|-----------|
| Client-side tool execution vs server-side | Client has direct filesystem access and keeps code local; server-side would require uploading the codebase |
| WebSocket vs SSE | WebSocket supports bidirectional communication (tool results flow back); SSE requires a separate POST channel |
| Prompt prefix caching | Reduces TTFT by 6x but dedicating 30% of GPU memory to KV-cache reduces max batch size by ~20% |

## 9. Closing (30s)
> A WebSocket-connected agentic loop where the Session Manager orchestrates multi-turn LLM inference and client-side tool execution. The LLM streams tokens until it emits tool calls, which execute locally on the user's machine with permission gates, then results feed back for the next inference round. Prompt prefix caching achieves 70-85% hit rates, reducing time-to-first-token from 2 seconds to 300ms. Context compression keeps conversations within window limits. The system scales through cache-aware routing, multi-model selection, and continuous batching on the GPU cluster.
