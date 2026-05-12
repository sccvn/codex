# ADR-006: Use OpenAI Responses API (Not Chat Completions)

## Status
Accepted

## Date
2026-05-12

## Context
Codex is built on OpenAI's model APIs. Two primary API surfaces are available:
1. **Chat Completions API** (`/v1/chat/completions`): message list in, message out; tool calls require manual state management
2. **Responses API** (`/v1/responses`): stateful conversation threads, server-side tool execution state, streaming events

The Codex agent loop needs:
- Streaming text output (token-by-token for low latency)
- Parallel tool call execution
- Stateful context management (conversation history, compaction)
- Reasoning token support (for o-series models)

## Decision
Build the agent loop on the OpenAI Responses API. The `ModelClientSession` in `codex-rs/core/src/client/` sends requests to `/v1/responses` and processes the `ResponseEvent` stream. The session tracks `previous_response_id` for stateful chaining.

## Alternatives Considered

### Chat Completions API
- Pros: More widely supported by third-party providers; simpler request format
- Cons: No server-side state — the client must reconstruct the full message list each turn (O(n) tokens per turn); tool call results must be manually injected; no streaming event types for tool execution lifecycle
- Rejected: The Responses API's server-side state tracking and richer streaming events are essential for the agent's performance profile

### LiteLLM / OpenAI-compatible proxy
- Pros: Provider-agnostic, can use non-OpenAI models
- Cons: Adds a dependency; not all providers support Responses API; compatibility layer may not expose reasoning tokens
- Rejected: Codex targets OpenAI models as the primary use case; provider abstraction can be added later

## Consequences
- `codex-rs/backend-client/` holds the OpenAI API client types
- Context compaction (`Op::Compact`) is essential because the Responses API still has a context window limit; auto-compact fires when token count exceeds `model_auto_compact_token_limit`
- `previous_response_id` chaining means the server holds conversation state; session resume requires reconstructing this chain from local history
- Reasoning tokens (`AgentReasoningEvent`) are only available on o-series models
