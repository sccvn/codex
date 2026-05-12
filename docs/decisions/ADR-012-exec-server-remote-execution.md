# ADR-012: Exec-Server for Remote and Isolated Command Execution

## Status
Accepted

## Date
2026-05-12

## Context
Codex's default model executes shell commands on the local machine. Some use cases require running commands in a different environment:
- **Remote development**: code lives on a remote server; commands should execute there
- **Cloud sandboxes**: commands execute in an isolated cloud VM, not the developer's laptop
- **VS Code Remote**: the agent runs locally but the workspace is on a remote container

A standalone command execution daemon is needed that can:
- Accept JSON-RPC connections (WebSocket or stdio)
- Execute commands with sandboxing
- Perform file operations (read/write/create/delete)
- Make HTTP requests on behalf of the agent

## Decision
Implement `codex-rs/exec-server/` as a standalone JSON-RPC daemon:
- `--listen ws://0.0.0.0:PORT` for WebSocket connections
- `--listen stdio` for stdin/stdout
- `--remote <URL> --executor-id <ID>` to register with a central executor registry
- `FileSystemSandboxContext` restricts file operations to allowed root paths
- Protocol: `initialize` → `exec` → streaming `exec/outputDelta` → `exec/exited`; plus `fs/*` and `http/request` methods

## Alternatives Considered

### SSH + shell scripts
- Pros: Universally available, no custom daemon required
- Cons: No structured RPC; no streaming output events; no file operation protocol; authentication is separate
- Rejected: Structured JSON-RPC output is required for the agent loop to process tool results

### Docker exec
- Pros: Built on existing container infrastructure
- Cons: Requires Docker on both ends; container lifecycle management is out of scope
- Rejected: Exec-server is lighter-weight and works without containers

### Integrating into app-server
- Pros: One fewer binary
- Cons: App-server manages agent sessions; exec-server is a pure execution primitive; conflating them couples session management with raw execution
- Rejected: Separation of concerns

## Consequences
- Exec-server is an optional component; most users never need it
- `codex-rs/execpolicy/` provides exec policy rules that exec-server enforces
- The remote executor registry endpoint is not yet public (see PRD Open Questions)
