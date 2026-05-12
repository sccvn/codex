# ADR-002: Submission Queue / Event Queue (SQ/EQ) Bidirectional Protocol

## Status
Accepted

## Date
2026-05-12

## Context
The Codex agent loop runs asynchronously: the model streams output, tool calls execute in parallel, and users can interrupt or approve operations mid-turn. Multiple clients (TUI, VS Code extension, CLI exec mode, app-server) need to communicate with the same underlying agent core.

The protocol must support:
- Clients submitting operations at any time (user messages, approvals, interrupts)
- The agent emitting a stream of events (text deltas, tool call start/end, approval requests)
- Async, non-blocking communication on both directions
- A typed, versioned contract so multiple client types can use the same core

## Decision
Implement a Submission Queue / Event Queue pattern defined entirely in `codex-rs/protocol/src/protocol.rs`:
- **SQ (Op enum, ~25 variants)**: Client → agent. `CodexThread::submit(op: Op)` pushes onto a Tokio channel; `submission_loop()` processes ops sequentially.
- **EQ (EventMsg enum, ~35 variants)**: Agent → client. Events are emitted through a broadcast channel; clients hold a receiver and process events independently.

The protocol module is the single source of truth for all inter-process and in-process communication shapes.

## Alternatives Considered

### Direct function calls / callbacks
- Pros: Simple, no serialization overhead for in-process use
- Cons: Tight coupling between client code and agent internals; difficult to serialize for IPC; cannot multiplex multiple clients
- Rejected: Multi-client support (TUI + app-server + CLI) requires decoupling

### gRPC streaming
- Pros: Cross-language, schema-first, bi-directional streaming
- Cons: Adds protobuf dependency; schema evolution is more rigid; overkill for single-host IPC; requires HTTP/2 transport
- Rejected: JSON-RPC over stdio/UDS is simpler and sufficient for local IPC

### Actor model (e.g., Actix)
- Pros: Built-in mailbox pattern, typed messages
- Cons: Adds framework dependency; Actix's macro-heavy API obscures control flow; migration path for protocol changes is unclear
- Rejected: Tokio channels provide the same pattern without a framework

## Consequences
- `Op` and `EventMsg` enums are the versioned contract; all clients must handle unknown variants gracefully
- In-process clients (TUI) use channels directly; external clients (VS Code) use the JSON-RPC app-server layer which maps between the two
- Adding new agent capabilities requires adding an `Op` variant (input) and one or more `EventMsg` variants (output) — clear extension points
- Protocol versioning is implicit in the binary; breaking changes require a new release
