# Claude Code (Real-Time AI Coding Assistant) -- Cheat Sheet

## Key Numbers
- 100M turns/day, ~3,500 QPS peak
- TTFT target: < 500ms (with prompt caching), ITL: < 50ms
- Prompt cache hit rate: 70-85% for active conversations
- Context window: up to 200K tokens, compress at 80%
- ~34,000 GPU-equivalents for Opus-class serving at peak

## Core Components
- **CLI Client**: Renders streaming text, executes tools locally, manages permissions
- **API Gateway**: Auth, rate limiting, WebSocket termination
- **Session Manager**: Owns the agentic loop -- assembles prompts, orchestrates tool calls, manages context
- **Inference Router**: Routes to optimal GPU pool based on model, cache locality, and load
- **Inference Engine**: Continuous batching, KV-cache, LoRA adapters on GPU clusters
- **Prompt Cache**: Prefix-based KV-cache reuse across conversation turns
- **Context Compressor**: Summarizes old turns when approaching window limits

## Architecture in One Sentence
WebSocket-connected agentic loop where the Session Manager orchestrates multi-turn LLM inference with client-side tool execution, backed by prompt-cached GPU inference with continuous batching.

## Top 3 Trade-offs
1. **Client-side vs server-side tools**: Client-side gives security and filesystem access but adds round-trip latency; chosen because users need local environment access
2. **Prompt caching vs batch size**: Dedicating GPU memory to KV-cache reduces max batch size by ~20%, but cuts TTFT by 4-5x for multi-turn conversations
3. **Multi-model routing vs simplicity**: Routing simple queries to Haiku saves 5-10x cost but risks misclassification frustrating users on complex tasks

## Scaling Story
- **10x**: Scale GPU pools linearly; prompt caching prevents 10x compute growth; move session state to Redis; add regional gateway PoPs
- **100x**: Multi-region inference; speculative decoding for 2-3x throughput; disaggregate prefill vs decode onto different GPU types; distill specialized models for common tasks

## Closing Statement
The system is an agentic loop over WebSocket: the LLM reasons, emits tool calls executed client-side, ingests results, and continues until the task is done. Prompt caching, continuous batching, and multi-model routing make it fast and cost-efficient at scale.
