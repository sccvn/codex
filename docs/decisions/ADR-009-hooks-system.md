# ADR-009: Event-Driven Hooks System for Non-Invasive Extension

## Status
Accepted

## Date
2026-05-12

## Context
Power users and enterprise operators need to customize Codex behavior without forking the codebase:
- Logging all tool calls to an audit system
- Blocking specific dangerous commands at the policy level
- Injecting additional context before every user prompt (e.g., project guidelines)
- Running cleanup scripts after tool execution

These customizations should not require modifying `codex-core` or the main agent loop.

## Decision
Implement an event-driven hooks system in `codex-rs/hooks/`. Hooks are shell scripts or executables configured in `config.toml` under `[hooks]`. Eight event types are supported:
1. `PreToolUse` — fires before every tool call; can block or modify arguments
2. `PostToolUse` — fires after tool call completes; can inject context
3. `PermissionRequest` — fires when a permission check occurs
4. `PreCompact` / `PostCompact` — fires around context compaction
5. `SessionStart` — fires once per session initialization
6. `UserPromptSubmit` — fires before each user turn is sent to the model
7. `Stop` — fires when the agent loop stops

Hook scripts receive a JSON payload on stdin and return a JSON response on stdout. `HookStartedEvent` and `HookCompletedEvent` are emitted for observability.

## Alternatives Considered

### Middleware pattern in the agent loop
- Pros: Type-safe, same process
- Cons: Requires Rust code; forces contributors to rebuild Codex; tight coupling to internals
- Rejected: External hook scripts allow customization without recompilation

### Lua scripting
- Pros: Embedded scripting, no subprocess overhead
- Cons: Adds Lua runtime dependency; most users know shell scripting, not Lua
- Rejected: Shell scripts are universally understood

### Webhooks (HTTP callbacks)
- Pros: Language-agnostic, network-accessible
- Cons: Adds network latency to every tool call; requires users to run an HTTP server
- Rejected: Local subprocess execution is faster and simpler for a desktop tool

## Consequences
- Hook scripts must be fast; slow hooks block the agent loop (no async hook execution)
- Hook failures emit events but do not crash the agent session by default
- `codex-rs/core/src/hook_runtime.rs` integrates hooks into the turn lifecycle
- `/hooks` slash command in TUI allows browsing and testing hook configurations
