# ADR-008: Model Context Protocol (MCP) as the Primary Extensibility Standard

## Status
Accepted

## Date
2026-05-12

## Context
Codex needs a way to expose external capabilities to the agent (web search, file indexing, database access, API integrations). Options:
1. Built-in tools only (shell, apply_patch, web_search)
2. A bespoke Codex plugin API
3. An industry standard protocol

The agent also needs to be usable as a tool provider from external agent frameworks (Claude Desktop, other OpenAI Agents SDK integrations).

## Decision
Adopt the Model Context Protocol (MCP) as the standard for tool extension. `codex-rs/codex-mcp/` implements:
- **Consumer side**: `McpConnectionManager` connects to external MCP servers via stdio transport (spawns subprocess) or StreamableHttp transport (connects to URL with bearer token)
- **Provider side**: `codex mcp-server` starts Codex as an MCP server, exposing its built-in tools to other MCP clients

Tool naming follows the `mcp__<server_name>__<tool_name>` convention to avoid collisions. When total MCP tool count exceeds `DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD` (100), tools switch to deferred exposure via a `tool_search` meta-tool to avoid context window bloat.

## Alternatives Considered

### Bespoke Codex plugin API (WASM)
- Pros: Fine-grained control; sandboxable
- Cons: Requires building and maintaining a plugin runtime; low ecosystem adoption; plugin authors need Codex-specific SDK
- Rejected: MCP already has growing ecosystem adoption (Claude, Cursor, Zed)

### OpenAI Function Calling conventions only
- Pros: Already supported by the model
- Cons: No standard for server discovery, authentication, or transport; each integration is bespoke
- Rejected: MCP provides transport + discovery + auth conventions that function calling alone does not

### HTTP REST with OpenAPI
- Pros: Universal
- Cons: No streaming; requires per-server client code generation; no standard for tool listing
- Rejected: MCP's tool listing and streaming tool results are better fits for the agent loop

## Consequences
- Users configure MCP servers in `config.toml` under `[mcp_servers.<name>]` with `command`, `args`, `env` (stdio) or `url`, `auth` (HTTP)
- `codex mcp list/add/remove/login/logout` CLI commands manage server configuration
- MCP tool calls emit `McpToolCallBeginEvent` and `McpToolCallEndEvent` for observability
- Elicitation requests (`elicitation/create`) from MCP servers trigger the standard approval flow
- The 100-tool threshold for deferred exposure prevents the system prompt from exceeding context limits
