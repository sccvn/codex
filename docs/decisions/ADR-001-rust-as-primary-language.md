# ADR-001: Rust as Primary Implementation Language

## Status
Accepted

## Date
2026-05-12

## Context
Codex is a locally-running coding agent that executes shell commands, applies file patches, manages sandboxed subprocesses, and serves multiple concurrent clients over IPC. It needs to:
- Operate with minimal memory footprint (runs alongside editors and build tools)
- Execute safely on macOS, Linux, and Windows without a runtime dependency
- Integrate deeply with OS-level sandboxing primitives (bubblewrap, landlock, seccomp, Seatbelt, restricted tokens)
- Ship as a single binary via npm install (no JVM, Python, or Go runtime required for end-users)
- Be safe from memory corruption in the process that holds open file handles and sandbox credentials

The codebase spans `codex-rs/` (~90 crates) plus a thin Node.js npm wrapper (`codex-cli/`) for distribution.

## Decision
Use Rust as the primary implementation language for all core logic (agent loop, sandboxing, TUI, app-server, exec-server, MCP integration). Use Node.js only for the npm distribution wrapper and to re-export the compiled binary.

## Alternatives Considered

### Go
- Pros: Fast compilation, good stdlib, cross-platform, single binary
- Cons: GC pauses affect latency-sensitive terminal rendering; no equivalent to Rust's ownership for safe concurrency; cgo required for OS sandbox primitives introduces complexity
- Rejected: Memory safety guarantees and zero-GC latency are important for a tool that drives terminal rendering at 60fps

### Python
- Pros: Rapid development, large ecosystem
- Cons: Requires Python runtime on user machines; GIL limits parallelism; subprocess/OS API integration is verbose; startup latency is too high for a CLI tool
- Rejected: Runtime dependency and performance characteristics are unsuitable

### C++
- Pros: Maximum performance, direct OS API access
- Cons: No memory safety without careful auditing; modern async C++ is complex; toolchain fragmentation across platforms
- Rejected: Rust provides equivalent performance with memory safety guarantees

### TypeScript / Node.js
- Pros: Team familiarity, npm ecosystem
- Cons: V8 adds ~50MB baseline memory; sandboxing Node child processes is complex; no native bubblewrap/Seatbelt bindings
- Rejected: The npm wrapper already uses Node; putting the agent loop there too would eliminate the performance advantage

## Consequences
- Binary ships as a compiled artifact distributed via npm; no runtime dependency for end-users
- Entire `codex-rs/` workspace uses Rust 1.93.0 with edition 2024
- `cargo-nextest` is the preferred test runner for parallelism
- Bazel is used in CI/release alongside Cargo for hermetic builds
- New contributors need Rust knowledge; compensated by comprehensive CLAUDE.md and memory files
