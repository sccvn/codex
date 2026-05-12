# ADR-004: Platform-Specific OS Sandboxing Instead of a Unified Cross-Platform Sandbox

## Status
Accepted

## Date
2026-05-12

## Context
Codex executes arbitrary shell commands on behalf of an AI model. These commands can read or write files, make network connections, and spawn child processes. Unrestricted execution poses serious security risks — commands could exfiltrate data, modify system files, or install malware.

A sandbox is required. The design space includes:
- A single cross-platform sandbox abstraction (e.g., containers, VMs)
- OS-native sandboxing mechanisms per platform
- No sandboxing (rely entirely on approval policies)

Available OS primitives:
- **Linux**: bubblewrap (namespace isolation), landlock (filesystem access control), seccomp (syscall filtering)
- **macOS**: Seatbelt (Apple's sandbox profiles, `/usr/bin/sandbox-exec`)
- **Windows**: Restricted token + private desktop

## Decision
Implement platform-specific sandboxing using the strongest available OS primitive on each platform. A unified `SandboxManager` in `codex-rs/sandboxing/src/manager.rs` routes to the correct backend:
- Linux: bubblewrap namespaces + landlock filesystem rules + seccomp (in `codex-rs/bwrap/` and `codex-rs/linux-sandbox/`)
- macOS: Apple Seatbelt profiles generated from `seatbelt_base_policy.sbpl` + `seatbelt_network_policy.sbpl`
- Windows: Restricted token + optional private desktop isolation

Three sandbox modes: `read-only` (default), `workspace-write`, `danger-full-access`.

## Alternatives Considered

### Docker / OCI containers
- Pros: Truly isolated, reproducible, cross-platform
- Cons: Requires Docker daemon; adds 500ms+ startup latency per command; does not work without Docker installed; file path mapping adds complexity
- Rejected: Startup latency is prohibitive for a conversational agent that runs many short-lived commands

### gVisor
- Pros: Strong isolation, Linux compatible
- Cons: Linux-only; kernel compatibility issues; startup overhead similar to containers
- Rejected: Not cross-platform; too heavy for interactive use

### Single cross-platform sandbox abstraction
- Pros: Simpler API, one code path
- Cons: Would need to use the weakest common denominator across platforms; wastes the stronger guarantees available on Linux (landlock, seccomp)
- Rejected: Platform-specific depth is more important than abstraction purity

### Policy-only (no OS enforcement)
- Pros: Cross-platform with no OS dependencies
- Cons: Any bug in policy evaluation means no sandbox at all; no defense in depth
- Rejected: Policy is the first line; OS enforcement is the second — both are required

## Consequences
- Each platform has independent sandbox code paths and tests
- `codex sandbox <os> -- <cmd>` CLI command enables manual sandbox testing
- `SandboxMode` is config-driven: `read-only` (default), `workspace-write`, `danger-full-access`
- Windows sandbox is `disabled` by default (restricted token is opt-in) — migration to secure-by-default is an open question (see PRD Open Questions)
- Integration tests that run under Seatbelt on macOS detect `CODEX_SANDBOX=seatbelt` and skip nested sandbox tests
