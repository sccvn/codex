# ADR-016: App-Server Operates Both In-Process and as an External Daemon

## Status
Accepted

## Date
2026-05-12

## Context
The TUI and VS Code extension both need to interact with the Codex agent core. Two operating models are needed:
1. **TUI embedded use**: the TUI process includes the app-server and agent core in the same process (no IPC overhead)
2. **VS Code / Electron use**: a separate `codex app-server` daemon runs persistently; the editor connects to it via stdio or UDS

The app-server code must work in both modes without duplication.

## Decision
The app-server is implemented as a library (`codex-rs/app-server/`) that can be instantiated either:
- In-process by the TUI (via `AppServerSession` facade in `codex-rs/tui/src/`)
- As a standalone binary (`codex app-server`) listening on stdio or UDS

Both modes use the same `MessageProcessor`, `ConnectionSessionState`, and RPC handler modules. The transport layer is abstracted behind `AppServerTransport` so the same session code handles both stdio (newline-delimited JSON) and WebSocket (over UDS).

`codex-rs/app-server-daemon/` handles the daemon lifecycle (start/stop/status) for the `codex app-server daemon` subcommands.

## Alternatives Considered

### Always require an external daemon
- Pros: Simpler architecture; one code path
- Cons: Adds IPC overhead for the TUI; requires the daemon to be running before the TUI starts; startup latency for casual users
- Rejected: In-process mode is important for TUI UX

### Separate codebases for TUI and app-server
- Pros: Clean separation
- Cons: Protocol drift; features must be implemented twice; maintenance burden doubles
- Rejected: The shared app-server library ensures consistency

## Consequences
- `codex app-server daemon start/stop/status` manages the daemon lifecycle
- The TUI instantiates the app-server inline when not connecting to an existing daemon
- Port/socket path conflicts are handled by `is_stale_socket_path()` in `codex-rs/uds/`
- `codex stdio-to-uds` bridges stdio-only clients to an existing daemon's UDS
