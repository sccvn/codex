# ADR-010: Plugin and Skill Marketplace System

## Status
Accepted

## Date
2026-05-12

## Context
Codex's extensibility has three tiers:
1. **Raw MCP servers**: low-level tool provider connections
2. **Skills**: curated instruction sets injected into the system prompt for specific task patterns
3. **Plugins**: packaged bundles of MCP servers + skills + hooks, distributed via a marketplace

Users need a discoverable, one-click way to extend Codex's capabilities (e.g., "add GitHub integration", "add Notion integration"). Manually configuring MCP servers and hooks is a barrier for non-technical users.

## Decision
Implement a marketplace-based plugin system:
- `codex-rs/core-plugins/` manages plugin lifecycle (install, remove, sync)
- Plugins are fetched from `openai-curated` (vetted by OpenAI) and `openai-bundled` (shipped with Codex) marketplaces
- Each plugin manifest defines MCP server names, app connector IDs, and skill presence
- Skills (`codex-rs/core-skills/`) are injected into the system prompt as tool definitions; `allow_implicit_invocation` flag enables automatic activation
- Plugin marketplace sync happens asynchronously at session start so it doesn't block the user

## Alternatives Considered

### Manual MCP server configuration only
- Pros: Simpler; no marketplace infrastructure
- Cons: Requires users to know server command lines and environment variables; no discovery
- Rejected: Poor UX for non-developer users

### VS Code extension marketplace
- Pros: Existing distribution infrastructure
- Cons: Requires a VS Code extension per integration; doesn't work in CLI or TUI mode
- Rejected: Codex is not VS Code-only

### npm packages as plugins
- Pros: Familiar to JavaScript developers
- Cons: Requires Node.js runtime; mixed language support (Codex core is Rust); versioning model doesn't match agent capabilities
- Rejected: Plugins must work in all surfaces (TUI, CLI, app-server) regardless of runtime

## Consequences
- `codex plugin` CLI command shows installed plugins and marketplace listings
- `/plugins` slash command in TUI manages plugins interactively
- Skills with unsatisfied MCP server dependencies prompt the user to install the required plugin
- Plugin installation writes to `$CODEX_HOME/plugins/`; plugins are loaded at session start
