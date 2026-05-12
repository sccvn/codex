# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the Codex platform.

ADRs capture significant technical decisions, their context, the alternatives considered, and the consequences. They explain *why* the system is built the way it is.

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](ADR-001-rust-as-primary-language.md) | Rust as Primary Implementation Language | Accepted |
| [ADR-002](ADR-002-sq-eq-protocol-pattern.md) | Submission Queue / Event Queue Bidirectional Protocol | Accepted |
| [ADR-003](ADR-003-multi-crate-workspace-bazel.md) | Multi-Crate Cargo Workspace with Bazel for CI/Release | Accepted |
| [ADR-004](ADR-004-platform-specific-sandboxing.md) | Platform-Specific OS Sandboxing | Accepted |
| [ADR-005](ADR-005-json-rpc-app-server.md) | JSON-RPC 2.0 App-Server for Client Integration | Accepted |
| [ADR-006](ADR-006-openai-responses-api.md) | Use OpenAI Responses API (Not Chat Completions) | Accepted |
| [ADR-007](ADR-007-ratatui-tui.md) | ratatui for Terminal User Interface | Accepted |
| [ADR-008](ADR-008-mcp-extensibility.md) | Model Context Protocol (MCP) as Primary Extensibility Standard | Accepted |
| [ADR-009](ADR-009-hooks-system.md) | Event-Driven Hooks System for Non-Invasive Extension | Accepted |
| [ADR-010](ADR-010-plugin-skill-marketplace.md) | Plugin and Skill Marketplace System | Accepted |
| [ADR-011](ADR-011-dual-auth-chatgpt-apikey.md) | Dual Authentication — ChatGPT OAuth and API Key | Accepted |
| [ADR-012](ADR-012-exec-server-remote-execution.md) | Exec-Server for Remote and Isolated Command Execution | Accepted |
| [ADR-013](ADR-013-native-rpitit-async-traits.md) | Native RPITIT for Async Traits Instead of #[async_trait] | Accepted |
| [ADR-014](ADR-014-insta-snapshot-testing-tui.md) | insta Snapshot Testing for TUI Regression Detection | Accepted |
| [ADR-015](ADR-015-codex-core-isolation-policy.md) | Policy to Resist Growing codex-core | Accepted |
| [ADR-016](ADR-016-in-process-vs-daemon-app-server.md) | App-Server Operates Both In-Process and as External Daemon | Accepted |
| [ADR-017](ADR-017-approval-policy-enum.md) | Granular Approval Policy Enum for Shell Command Safety | Accepted |
| [ADR-018](ADR-018-session-persistence-resume-fork.md) | Session Persistence via Local History for Resume and Fork | Accepted |

## Adding New ADRs

1. Copy the template from any existing ADR
2. Number sequentially (ADR-019, ADR-020, ...)
3. Set status to `Proposed`; change to `Accepted` when the decision is finalized
4. Add a row to the index table above
5. If superseding an old ADR, update the old ADR's status to `Superseded by ADR-XXX`
