# ADR-003: Multi-Crate Cargo Workspace with Bazel for CI/Release

## Status
Accepted

## Date
2026-05-12

## Context
Codex started as a single Rust crate and grew to ~90 crates covering agent core, TUI, CLI, sandboxing, MCP, hooks, plugins, app-server, exec-server, and supporting utilities. Key concerns:
- Compilation times: a monolithic crate forces full recompilation on any change
- Dependency isolation: sandboxing code should not depend on TUI; protocol types should not depend on networking
- Incremental CI: only changed crates and their dependents should be rebuilt
- Release builds must be hermetic (reproducible, auditable)

## Decision
Organize all Rust code as a Cargo workspace in `codex-rs/` with individual crates, all named with the `codex-` prefix. Use Cargo for local development (incremental compilation, fast iteration). Use Bazel (with `rules_rust`) for CI and release builds (hermetic, parallel, cache-friendly).

The `just` task runner (`codex-rs/justfile`) provides ergonomic commands for both systems: `just fmt`, `just fix`, `just test`, `just bazel-lock-update`.

## Alternatives Considered

### Single monolithic crate
- Pros: Simpler dependency management, no inter-crate API design overhead
- Cons: Full recompilation on any change; cannot run `cargo test -p codex-tui` without building everything; cannot enforce dependency isolation
- Rejected: Build times become prohibitive as the codebase grows

### Cargo only (no Bazel)
- Pros: Simpler toolchain, no Bazel learning curve
- Cons: Cargo builds are not hermetic; cannot guarantee reproducible release artifacts across machines; limited remote caching
- Rejected: Release infrastructure requires hermeticity

### Bazel only (no Cargo)
- Pros: Single build system
- Cons: Bazel's Rust support is mature but slower to iterate than Cargo for development; most Rust tooling (rust-analyzer, clippy, cargo-insta) integrates natively with Cargo
- Rejected: Developer experience suffers significantly

## Consequences
- All crate names are prefixed `codex-` (e.g., `codex-core`, `codex-tui`, `codex-mcp`)
- `codex-core` has grown large; explicit policy to resist adding new code there (see ADR-015)
- After any `Cargo.toml` / `Cargo.lock` change, `just bazel-lock-update` and `just bazel-lock-check` must be run to keep `MODULE.bazel.lock` in sync
- Bazel `BUILD.bazel` files must be updated when adding `include_str!`, `include_bytes!`, or `sqlx::migrate!` calls that read source files at build time
