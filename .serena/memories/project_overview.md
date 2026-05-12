# Codex Project Overview

## Purpose
Codex is OpenAI's coding agent CLI tool that runs locally. It provides an interactive terminal UI (TUI), CLI, MCP server, and app-server for AI-assisted software development. Users can sign in with ChatGPT or use an API key.

## Repository Structure
- `codex-rs/` — Primary Rust workspace (main backend, TUI, CLI binary, all core logic)
- `codex-cli/` — Minimal Node.js/npm wrapper package (`@openai/codex`)
- `sdk/` — SDKs (Python SDK in `sdk/python`)
- `docs/` — Documentation
- `scripts/` — Build/release scripts
- `third_party/` — Third-party dependencies
- `.codex/` — Codex agent config
- `.github/` — CI/CD workflows

## Tech Stack
- **Primary language**: Rust (toolchain: 1.93.0)
- **Build systems**: Cargo (primary dev), Bazel (CI/release)
- **Test runner**: `cargo-nextest` (preferred over `cargo test`)
- **TUI framework**: ratatui
- **Async runtime**: Tokio
- **Task runner**: `just` (justfile located at `codex-rs/justfile`, sets working-directory to `codex-rs`)
- **Python**: Used for Python SDK and tooling (uv package manager)
- **Package manager (Node)**: pnpm (workspace)

## Main Entry Points
- `codex` binary — CLI and interactive TUI
- MCP server (`codex-mcp-server`)
- App server (`codex-app-server`)
- `codex exec-server` — exec server for TUI

## Key Crates (codex-rs workspace)
All crate names are prefixed with `codex-`. Notable ones:
- `codex-core` — Core logic (large crate, avoid adding to it)
- `codex-tui` — Terminal UI (ratatui-based)
- `codex-cli` — CLI binary
- `codex-mcp` — MCP connection manager
- `codex-mcp-server` — MCP server
- `codex-app-server` — App server
- `codex-app-server-protocol` — App server protocol (v2 is active)
- `codex-protocol` — Protocol types
- `codex-exec` — Execution engine
- `codex-sandboxing` — Sandbox support
- `codex-config` — Configuration
- `codex-login` — Authentication
