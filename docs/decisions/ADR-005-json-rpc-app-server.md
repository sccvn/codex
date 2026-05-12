# ADR-005: JSON-RPC 2.0 App-Server for Client Integration

## Status
Accepted

## Date
2026-05-12

## Context
Multiple client surfaces need to interact with Codex: the built-in TUI, a VS Code extension, a desktop Electron app, and future third-party integrations. Options for how these clients interact with the Codex agent core:
1. Link the core as a library in each client
2. Expose the core via a network protocol (HTTP, gRPC, WebSocket)
3. Expose the core via a local IPC protocol (stdio, Unix Domain Socket)

Requirements:
- TypeScript clients (VS Code extension, Electron) need a protocol with type generation
- The TUI (Rust) needs low-latency in-process access
- The daemon mode must be startable by VS Code without user interaction
- Protocol must support both request-response (RPC) and server-push (notifications)

## Decision
Implement a JSON-RPC 2.0 app-server (`codex-rs/app-server/`) that listens on:
- **stdio** (newline-delimited JSON): for VS Code extension using `stdio-to-uds` bridge
- **Unix Domain Socket** (WebSocket framing): for Electron desktop app and TUI daemon mode

The app-server v2 protocol (`codex-rs/app-server-protocol/src/protocol/v2/`) defines 20+ RPC method modules. TypeScript bindings are auto-generated via `just write-app-server-schema`. The TUI can use the app-server in-process (no network hop) or connect to an external daemon.

## Alternatives Considered

### Library linking only
- Pros: Zero serialization overhead; simplest for Rust clients
- Cons: TypeScript clients (VS Code) cannot link Rust libraries natively; requires N implementations of the same protocol for N client types
- Rejected: TypeScript/Electron clients are a first-class target

### HTTP REST API
- Pros: Universal, easy to test with curl
- Cons: Request-response only — cannot push events (tool execution, text deltas) without polling or SSE; polling adds latency
- Rejected: Streaming events are a core requirement

### gRPC
- Pros: Schema-first, code generation, bidirectional streaming
- Cons: protobuf adds dependency; HTTP/2 transport is heavier than UDS for local IPC; TypeScript gRPC clients are more complex than JSON
- Rejected: JSON-RPC over UDS has lower overhead for single-host use

### Named pipes (Windows) / FIFO
- Pros: Simple, cross-platform for streams
- Cons: Half-duplex; no framing; message multiplexing requires custom protocol
- Rejected: WebSocket over UDS provides full-duplex + framing + message ordering

## Consequences
- v2 protocol is camelCase on the wire (`#[serde(rename_all = "camelCase")]`), except config endpoints which mirror `config.toml` snake_case keys
- All v2 types have `#[ts(export_to = "v2/")]` for TypeScript generation
- v1 app-server API is frozen; all new development in v2 only
- `codex-rs/uds/` provides cross-platform UDS support including Windows (via `uds_windows` crate)
- `codex-rs/stdio-to-uds/` bridges stdio-only clients (e.g., VS Code language server protocol) to UDS
