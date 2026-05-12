# PRD: Codex Platform — Full Product Overview

> **Audience:** AI agents implementing features in this codebase.
> **Scope:** As-is documentation of all surfaces — CLI, TUI, MCP server, app-server, exec-server, sandboxing, hooks, plugins, skills, desktop app.
> **Depth:** Granular user stories down to individual crate/module responsibilities.

---

## Introduction

Codex is OpenAI's locally-running coding agent. It accepts natural-language instructions from users and autonomously executes multi-step software engineering tasks: writing code, running shell commands, applying patches, searching the web, reading files, and calling external tools — all while respecting configurable safety boundaries.

The platform is written primarily in Rust (`codex-rs/` workspace, ~90 crates) with a minimal Node.js CLI wrapper (`codex-cli/`) for npm distribution. It exposes multiple interaction surfaces:

- **Interactive TUI** — full-screen ratatui terminal UI with chat composer, transcript viewer, approval modals, slash commands, and voice input
- **Non-interactive CLI** (`codex exec`) — single-shot agent execution for scripting/CI
- **App-server** — JSON-RPC 2.0 daemon that powers VS Code extensions and desktop apps
- **MCP server** (`codex mcp-server`) — exposes Codex as a Model Context Protocol tool provider
- **Exec-server** — standalone sandboxed command-execution daemon for remote/isolated environments

The agent loop is built on OpenAI's Responses API. Users can authenticate via ChatGPT login (OAuth device code) or API key. All shell command execution is sandboxed on Linux (bubblewrap + landlock + seccomp), macOS (Seatbelt), and Windows (restricted token).

---

## Goals

- Provide an autonomous, interactive coding agent that runs entirely on the user's machine
- Allow the model to read/write files, run shell commands, and call external MCP tools with configurable safety guardrails
- Support multiple interaction surfaces (TUI, CLI, desktop app, VS Code extension) through a unified app-server JSON-RPC protocol
- Enable extensibility via MCP servers, hooks, plugins, and skills without modifying core code
- Enforce platform-appropriate sandboxing (Linux/macOS/Windows) so agent actions cannot exceed the granted permission profile
- Allow session persistence: users can resume, fork, and compact previous conversations
- Support multi-agent workflows: spawn sub-agents, switch between agent threads, monitor progress across threads

---

## User Stories

Stories are grouped by subsystem (crate). Each story is scoped to be implementable in one focused session.

---

### SUBSYSTEM 1 — Core Agent Loop (`codex-core`, `codex-protocol`)

#### US-001: Submit a user turn to the agent
**Description:** As a client (TUI or app-server), I want to submit a user message (`Op::UserTurn`) to a `CodexThread` so that the agent loop starts and produces a response stream.

**Acceptance Criteria:**
- [ ] `CodexThread::submit(Op::UserTurn { items, cwd, approval_policy, model, ... })` sends the operation to the `submission_loop` channel in `codex-rs/core/src/session/mod.rs`
- [ ] `submission_loop()` in `codex-rs/core/src/session/handlers.rs` dispatches `Op::UserTurn` to `user_input_or_turn()`
- [ ] A `TurnStartedEvent` is emitted before any model call
- [ ] `run_turn()` in `codex-rs/core/src/session/turn.rs` begins streaming model responses via `ModelClientSession`
- [ ] A `TurnCompleteEvent` is emitted when the model finishes
- [ ] Tests pass: `cargo test -p codex-core`

#### US-002: Stream agent text output to the client
**Description:** As a client, I want to receive `AgentMessageEvent` deltas in real time so that the UI can render the agent's response incrementally.

**Acceptance Criteria:**
- [ ] `EventMsg::AgentMessage(AgentMessageEvent)` is sent for each text chunk from the model stream
- [ ] `EventMsg::AgentReasoning(AgentReasoningEvent)` is sent separately for reasoning/think tokens
- [ ] Events are forwarded to the client's event receiver without buffering the full response
- [ ] Tests pass: `cargo test -p codex-core`

#### US-003: Execute a shell tool call with sandbox
**Description:** As the agent, I want to call the `shell` tool so that I can run commands in a sandboxed environment and return the output to the model.

**Acceptance Criteria:**
- [ ] Model output with a `shell` tool call triggers `process_exec_tool_call()` in `codex-rs/core/src/exec.rs`
- [ ] `build_exec_request()` selects sandbox type from `SandboxManager` based on current `SandboxMode`
- [ ] `ExecCommandBeginEvent`, streaming `ExecCommandOutputDeltaEvent`s, and `ExecCommandEndEvent` are emitted
- [ ] Output (stdout + stderr + exit code) is returned to the model as the tool call result
- [ ] Execution is bounded by `DEFAULT_EXEC_COMMAND_TIMEOUT_MS` (configurable)
- [ ] Output is capped at `EXEC_OUTPUT_MAX_BYTES`
- [ ] Tests pass: `cargo test -p codex-core`

#### US-004: Request and process shell command approval
**Description:** As the agent, I want to request user approval before running a shell command when the approval policy requires it, so that humans can review dangerous operations.

**Acceptance Criteria:**
- [ ] `create_exec_approval_requirement_for_command()` in `codex-rs/core/src/exec_policy.rs` evaluates `AskForApproval` policy
- [ ] When approval is required, `ExecApprovalRequestEvent` is emitted with the command and working directory
- [ ] The agent loop waits for `Op::ExecApproval { id, decision }` before proceeding
- [ ] `ReviewDecision::Approved` → command executes; `Denied` → error returned to model; `Abort` → turn interrupted
- [ ] `ReviewDecision::ApprovedExecpolicyAmendment` → exec policy rules are updated in-process
- [ ] Tests pass: `cargo test -p codex-core`

#### US-005: Apply a code patch with approval
**Description:** As the agent, I want to use the `apply_patch` tool to write file changes, optionally requesting approval first, so that code edits are safe and reviewable.

**Acceptance Criteria:**
- [ ] Model tool call to `apply_patch` triggers `ApplyPatchApprovalRequestEvent` when policy requires it
- [ ] Client responds with `Op::PatchApproval { id, decision }`
- [ ] Patch is applied atomically: `PatchApplyBeginEvent` → `PatchApplyUpdatedEvent`(s) → `PatchApplyEndEvent`
- [ ] File writes are confined to paths permitted by the active `PermissionProfile`
- [ ] Tests pass: `cargo test -p codex-core`

#### US-006: Compact a conversation when context limit approaches
**Description:** As the system, I want to automatically or manually summarize a long conversation to prevent hitting the model's context window limit.

**Acceptance Criteria:**
- [ ] `Op::Compact` triggers `run_inline_auto_compact_task()` in `codex-rs/core/src/compact.rs`
- [ ] Model is prompted to summarize the conversation history
- [ ] `ContextCompactedEvent` is emitted on success
- [ ] Auto-compact triggers when token count exceeds `model_auto_compact_token_limit` from config
- [ ] Compact remote v2 variant (`compact_remote_v2.rs`) works for remote model clients
- [ ] Tests pass: `cargo test -p codex-core`

#### US-007: Roll back conversation turns
**Description:** As a user, I want to undo the last N turns of a conversation so that I can retry from an earlier state.

**Acceptance Criteria:**
- [ ] `Op::ThreadRollback { num_turns }` drops the last N turns from in-memory context in `session/handlers.rs:488`
- [ ] `ThreadRolledBackEvent` is emitted confirming the rollback
- [ ] After rollback, the next user turn starts from the rolled-back state
- [ ] Tests pass: `cargo test -p codex-core`

#### US-008: Manage session lifecycle (start, resume, fork, shutdown)
**Description:** As the system, I want session lifecycle operations (start, resume, fork, shutdown) to correctly initialize and tear down all session state.

**Acceptance Criteria:**
- [ ] `ThreadManager::new_thread()` creates a fresh `CodexThread` with `SessionConfiguration` (model, cwd, permissions, approval policy)
- [ ] `ThreadManager::fork_thread()` creates a new thread that inherits conversation history up to the fork point
- [ ] `Op::Shutdown` sends `ShutdownComplete` event and closes the `submission_loop`
- [ ] `SessionConfiguredEvent` is the first event emitted on any new session, containing model and config details
- [ ] Tests pass: `cargo test -p codex-core`

#### US-009: Handle MCP tool calls
**Description:** As the agent, I want to call tools provided by connected MCP servers so that I can use external capabilities (file search, browser, Slack, etc.).

**Acceptance Criteria:**
- [ ] MCP tool calls from the model stream are routed to the correct `McpConnectionManager` client in `codex-rs/codex-mcp/src/mcp_connection_manager.rs`
- [ ] `McpToolCallBeginEvent` and `McpToolCallEndEvent` are emitted
- [ ] Approval flow fires `ElicitationRequest` when the MCP server uses `elicitation/create`
- [ ] `mcp_permission_prompt_is_auto_approved()` respects `AskForApproval` policy and `PermissionProfile`
- [ ] Tool results are returned to the model correctly
- [ ] Tests pass: `cargo test -p codex-core`

#### US-010: Web search tool call
**Description:** As the agent, I want to call the `web_search` tool to retrieve live information from the internet.

**Acceptance Criteria:**
- [ ] `WebSearchBeginEvent` and `WebSearchEndEvent` are emitted around web search execution
- [ ] Web search is only available when `--search` flag is passed or config enables it
- [ ] Search results are returned to the model as tool output
- [ ] Tests pass: `cargo test -p codex-core`

---

### SUBSYSTEM 2 — Protocol Types (`codex-protocol`)

#### US-011: Define the Op enum (client → agent operations)
**Description:** As a protocol consumer, I want a complete, versioned `Op` enum so that all client-to-agent operations are type-safe and self-documenting.

**Acceptance Criteria:**
- [ ] `Op` enum is defined in `codex-rs/protocol/src/protocol.rs` and covers: `UserTurn`, `UserInput`, `UserInputWithTurnContext`, `OverrideTurnContext`, `ExecApproval`, `PatchApproval`, `UserInputAnswer`, `RequestPermissionsResponse`, `DynamicToolResponse`, `ResolveElicitation`, `Interrupt`, `CleanBackgroundTerminals`, `Compact`, `ThreadRollback`, `SetThreadMemoryMode`, `Review`, `Shutdown`, `RunUserShellCommand`, and all `Realtime*` variants
- [ ] All variants have full serde serialization/deserialization
- [ ] Tests pass: `cargo test -p codex-protocol`

#### US-012: Define the EventMsg enum (agent → client events)
**Description:** As a protocol consumer, I want a complete `EventMsg` enum so that all agent-to-client events are type-safe.

**Acceptance Criteria:**
- [ ] `EventMsg` covers all lifecycle, output, approval, tool execution, and status events (full list in `codex-rs/protocol/src/protocol.rs` lines 1262-1449)
- [ ] All event structs serialize/deserialize correctly with serde
- [ ] Tests pass: `cargo test -p codex-protocol`

---

### SUBSYSTEM 3 — TUI (`codex-tui`)

#### US-013: Render the main chat transcript
**Description:** As a user, I want to see the full conversation history rendered in the terminal so that I can read previous messages, tool calls, and outputs.

**Acceptance Criteria:**
- [ ] `ChatWidget` in `codex-rs/tui/src/chatwidget.rs` renders agent messages, user messages, reasoning, exec output, and patch diffs
- [ ] New content appends without re-rendering the full history
- [ ] Transcript is scrollable via `Ctrl+T` overlay
- [ ] Markdown is rendered with syntax highlighting (from `codex-rs/tui/src/markdown_render.rs`)
- [ ] Snapshot tests in `codex-rs/tui/tests/` pass; run `cargo test -p codex-tui`

#### US-014: Compose and submit user messages
**Description:** As a user, I want a multi-line text input composer at the bottom of the screen so that I can write and submit messages to the agent.

**Acceptance Criteria:**
- [ ] `ChatComposer` in `codex-rs/tui/src/bottom_pane/chat_composer.rs` supports multi-line input
- [ ] `Enter` submits the current draft
- [ ] `Tab` queues the draft while a task is running
- [ ] Emacs-style keybindings work (Ctrl+A, Ctrl+E, Ctrl+K, Ctrl+U, Ctrl+Y, etc.)
- [ ] Vim mode is toggled via `/vim` slash command or config
- [ ] Snapshot tests pass; run `cargo test -p codex-tui`

#### US-015: Display and respond to shell approval requests
**Description:** As a user, I want an approval modal to appear when the agent requests permission to run a shell command so that I can review and approve/deny it.

**Acceptance Criteria:**
- [ ] `PendingThreadApprovals` in `codex-rs/tui/src/bottom_pane/` renders command with working directory
- [ ] Keyboard shortcuts work: `y` (approve once), `a` (approve for session), `p` (approve for prefix), `d` (deny), `n`/Esc (decline)
- [ ] `Ctrl+A` opens fullscreen approval overlay
- [ ] Decision is sent as `Op::ExecApproval` to the agent
- [ ] Snapshot tests pass; run `cargo test -p codex-tui`

#### US-016: Slash command dispatch
**Description:** As a user, I want to type `/` in the composer to invoke slash commands so that I can control TUI behavior without leaving the chat interface.

**Acceptance Criteria:**
- [ ] Typing `/` shows `CommandPopup` with all available commands and descriptions
- [ ] All commands in `codex-rs/tui/src/slash_command.rs` are dispatched correctly (full list: `/model`, `/ide`, `/permissions`, `/keymap`, `/vim`, `/new`, `/resume`, `/fork`, `/compact`, `/plan`, `/diff`, `/review`, `/status`, `/memories`, `/skills`, `/hooks`, `/mcp`, `/debug-config`, `/title`, `/statusline`, `/theme`, `/personality`, `/settings`, `/quit`, `/logout`, `/feedback`, `/collab`, `/agent`, `/side`, `/goal`, `/raw`, `/copy`, `/mention`, `/ps`, `/stop`, `/init`, `/realtime`, `/approve`, `/setup-default-sandbox`, `/sandbox-add-read-dir`, `/experimental`, `/plugins`, `/apps`, `/rollout`)
- [ ] Commands with inline args (e.g., `/rename <name>`) parse arguments correctly via `PromptArgs`
- [ ] Snapshot tests pass; run `cargo test -p codex-tui`

#### US-017: Session resume picker
**Description:** As a user, I want to browse and resume previous conversations so that I can continue work across terminal sessions.

**Acceptance Criteria:**
- [ ] `--resume-picker` flag shows interactive session list with cwd, date, and summary
- [ ] `--resume-last` resumes the most recent session without prompting
- [ ] `--resume-session-id <UUID>` resumes a specific session by ID
- [ ] If the resumed session's cwd differs from current, user is prompted to choose
- [ ] `/resume [SESSION_ID]` slash command works in-session
- [ ] Tests pass: `cargo test -p codex-tui`

#### US-018: Multi-agent thread navigation
**Description:** As a user running multiple agent threads, I want to switch between active agent threads using keyboard shortcuts so that I can monitor parallel work.

**Acceptance Criteria:**
- [ ] `Alt+Left` / `Alt+Right` cycles between active agent threads
- [ ] `/agent` or `/subagents` shows picker with agent names, roles, and status
- [ ] Closed/finished threads are shown dimmed
- [ ] Active thread transcript is rendered in the main view
- [ ] Tests pass: `cargo test -p codex-tui`

#### US-019: Voice / realtime input
**Description:** As a user, I want to use voice input via the `/realtime` command so that I can speak to the agent instead of typing.

**Acceptance Criteria:**
- [ ] `/realtime` toggles realtime voice mode (guarded as experimental)
- [ ] `VoiceCapture` in `codex-rs/tui/src/voice.rs` captures audio at 24,000 Hz mono
- [ ] `RecordingMeterState` renders a live level meter with Braille characters
- [ ] `/settings` allows selecting audio input device
- [ ] `Op::RealtimeConversationStart`, `Op::RealtimeConversationAudio`, `Op::RealtimeConversationClose` are submitted correctly
- [ ] Tests pass: `cargo test -p codex-tui`

#### US-020: TUI onboarding flow
**Description:** As a new user, I want a guided onboarding flow when I first launch Codex so that I can authenticate and configure basic settings before starting a conversation.

**Acceptance Criteria:**
- [ ] Onboarding state machine in `codex-rs/tui/src/onboarding/` runs: Welcome → Auth → Trust Directory
- [ ] Auth step supports ChatGPT OAuth device code flow and API key entry
- [ ] API key field disables accidental quit (treats `q` as text input when field is non-empty)
- [ ] Trust directory step prompts for `~` or cwd trust grant
- [ ] On completion, transitions to main chat UI
- [ ] Tests pass: `cargo test -p codex-tui`

#### US-021: Model selection and reasoning effort
**Description:** As a user, I want to change the active model and its reasoning effort in-session so that I can balance speed vs. quality for different tasks.

**Acceptance Criteria:**
- [ ] `/model` slash command opens model picker showing all available models from `ModelCatalog`
- [ ] Reasoning effort adjustable via `Alt+,` (decrease) and `Alt+.` (increase)
- [ ] Selected model and effort are stored in `SessionConfiguration` and sent with the next `Op::UserTurn`
- [ ] Status line reflects current model name
- [ ] Tests pass: `cargo test -p codex-tui`

---

### SUBSYSTEM 4 — CLI (`codex-cli`)

#### US-022: Interactive TUI launch (`codex` with no subcommand)
**Description:** As a user, I want running `codex` (no subcommand) to launch the full interactive TUI so that I can start a conversation immediately.

**Acceptance Criteria:**
- [ ] `main()` in `codex-rs/cli/src/main.rs` with no subcommand calls `run_interactive_tui()`
- [ ] Optional `PROMPT` positional argument pre-fills the first user message
- [ ] `--no-alt-screen` flag runs inline (no alternate screen, compatible with Zellij)
- [ ] `--ask-for-approval <MODE>` overrides config approval policy
- [ ] `--dangerously-bypass-approvals-and-sandbox` bypasses all sandbox/approval checks
- [ ] Tests pass: `cargo test -p codex-cli`

#### US-023: Non-interactive exec mode (`codex exec`)
**Description:** As a developer/CI system, I want to run `codex exec "prompt"` to execute a single agent turn non-interactively and receive structured output on stdout.

**Acceptance Criteria:**
- [ ] `Subcommand::Exec` (alias `e`) routes to `codex_exec` crate handler
- [ ] Agent executes prompt, runs tools, and exits when done
- [ ] Exit code reflects success/failure
- [ ] No TUI rendering; output goes directly to stdout
- [ ] Tests pass: `cargo test -p codex-cli`

#### US-024: Authentication commands (`codex login` / `codex logout`)
**Description:** As a user, I want `codex login` and `codex logout` CLI commands so that I can manage authentication from the terminal.

**Acceptance Criteria:**
- [ ] `codex login` supports subcommands: device code OAuth, API key input, workspace login
- [ ] `codex logout` removes stored credentials from the configured `AuthCredentialsStoreMode` (file/keyring/auto)
- [ ] Credentials are stored at `$CODEX_HOME/auth.json` (file mode) or OS keyring (keyring mode)
- [ ] Tests pass: `cargo test -p codex-cli`

#### US-025: MCP server management (`codex mcp`)
**Description:** As a user, I want CLI commands to list, add, remove, and authenticate MCP servers so that I can manage external tool integrations.

**Acceptance Criteria:**
- [ ] `codex mcp list` prints all configured MCP servers with status
- [ ] `codex mcp get <name>` prints details for a specific server
- [ ] `codex mcp add` adds a new server to config
- [ ] `codex mcp remove <name>` removes a server from config
- [ ] `codex mcp login <name>` initiates OAuth flow for a server
- [ ] `codex mcp logout <name>` removes OAuth credentials
- [ ] Tests pass: `cargo test -p codex-cli`

#### US-026: Sandbox testing command (`codex sandbox`)
**Description:** As a developer, I want `codex sandbox <os> -- <command>` to run a command inside the platform sandbox so that I can test sandbox behavior.

**Acceptance Criteria:**
- [ ] `codex sandbox macos -- ls` runs under Apple Seatbelt
- [ ] `codex sandbox linux -- ls` runs under Linux bubblewrap/landlock
- [ ] `codex sandbox windows -- ls` runs under Windows restricted token
- [ ] `--permissions-profile <NAME>` selects the permission profile
- [ ] `--log-denials` captures sandbox denial logs (macOS)
- [ ] Tests pass: `cargo test -p codex-cli`

#### US-027: App-server management (`codex app-server`)
**Description:** As an operator, I want `codex app-server` subcommands to start, stop, and inspect the app-server daemon.

**Acceptance Criteria:**
- [ ] `codex app-server --listen stdio://` starts app-server on stdio
- [ ] `codex app-server --listen unix://` starts app-server on UDS at default socket path
- [ ] `codex app-server daemon start` launches the daemon in background
- [ ] `codex app-server daemon stop` terminates the daemon
- [ ] `codex app-server daemon status` reports daemon running/stopped
- [ ] `codex app-server generate-ts` emits TypeScript bindings for the v2 protocol
- [ ] `codex app-server generate-json-schema` emits JSON Schema bundle
- [ ] Tests pass: `cargo test -p codex-cli`

---

### SUBSYSTEM 5 — Configuration (`codex-config`)

#### US-028: Load and merge configuration layers
**Description:** As the system, I want configuration to be loaded from multiple layers (global, workspace, environment overrides) and merged in priority order so that users can have project-specific settings.

**Acceptance Criteria:**
- [ ] `loader.rs` in `codex-rs/config/src/` loads `~/.config/codex/config.toml` (global) and `.codex/config.toml` (workspace)
- [ ] Workspace config takes precedence over global config
- [ ] CLI flags override all config layers
- [ ] `ConfigToml` struct in `codex-rs/config/src/config_toml.rs` covers: `model`, `approval_policy`, `sandbox_mode`, `mcp_servers`, `instructions`, `permissions`, `file_opener`, `tui`, `history`, `windows`, and all other documented fields
- [ ] `just write-config-schema` regenerates `codex-rs/core/config.schema.json` correctly
- [ ] Tests pass: `cargo test -p codex-config`

#### US-029: Permission profiles
**Description:** As an operator, I want named permission profiles (filesystem + network rules) that can be selected per-session so that different projects can have different security policies.

**Acceptance Criteria:**
- [ ] `PermissionsToml` in `codex-rs/config/src/permissions_toml.rs` defines `FilesystemPermissionsToml` and `NetworkToml`
- [ ] `FilesystemPermissionToml` supports `Access(FileSystemAccessMode)` and `Scoped(...)` variants
- [ ] `NetworkToml` supports `enabled`, `proxy_url`, domain allow/deny rules, and Unix socket permissions
- [ ] Multiple named profiles can be defined under `[permissions.<name>]` in config
- [ ] `default_permissions` config key selects the active profile
- [ ] Tests pass: `cargo test -p codex-config`

#### US-030: Exec policy rules
**Description:** As an operator, I want to define exec policy rules in `.codex/rules/*.rules` files so that specific commands are automatically allowed, prompted, or forbidden.

**Acceptance Criteria:**
- [ ] Rules files are discovered from `.codex/rules/` directory with `.rules` extension
- [ ] `PrefixRule` matches command prefix patterns (`PatternToken::Single` and `PatternToken::Alts`)
- [ ] `Decision` variants: `Allow`, `Prompt`, `Forbidden` work correctly
- [ ] `NetworkRule` supports protocols: `http`, `https`, `https_connect`, `socks5_tcp`, `socks5_udp`
- [ ] Tests pass: `cargo test -p codex-execpolicy`

---

### SUBSYSTEM 6 — Authentication (`codex-login`)

#### US-031: ChatGPT OAuth device code login
**Description:** As a user, I want to log in via ChatGPT OAuth so that I can use Codex as part of my ChatGPT plan without managing API keys.

**Acceptance Criteria:**
- [ ] Device code flow initiates in `codex-rs/login/src/auth/manager.rs`
- [ ] User is shown a device code URL and prompted to open it in a browser
- [ ] On completion, access token + refresh token are stored per `AuthCredentialsStoreMode`
- [ ] Token auto-refresh runs within the 8-hour window
- [ ] `codex logout` calls token revocation endpoint
- [ ] Tests pass: `cargo test -p codex-login`

#### US-032: API key authentication
**Description:** As a developer, I want to authenticate with an OpenAI or Codex API key so that I can use Codex with programmatic access.

**Acceptance Criteria:**
- [ ] `OPENAI_API_KEY` environment variable is read and used as `CodexAuth::ApiKeyAuth`
- [ ] `CODEX_API_KEY` is also supported
- [ ] API key is stored in keyring or file per `cli_auth_credentials_store` config
- [ ] Authentication mode is reflected in `SessionConfiguredEvent`
- [ ] Tests pass: `cargo test -p codex-login`

---

### SUBSYSTEM 7 — Sandboxing (`codex-sandboxing`, `codex-linux-sandbox`, `codex-execpolicy`)

#### US-033: Linux sandbox via bubblewrap + landlock
**Description:** As the system on Linux, I want shell commands executed by the agent to run inside a bubblewrap namespace with landlock filesystem restrictions so that they cannot access unauthorized paths.

**Acceptance Criteria:**
- [ ] `SandboxManager` in `codex-rs/sandboxing/src/manager.rs` selects bubblewrap/landlock for Linux
- [ ] Bubblewrap namespace isolates: mount, pid, net, user namespaces
- [ ] Landlock rules restrict filesystem access to the `PermissionProfile`'s allowed paths
- [ ] Seccomp filter is applied with `no_new_privs`
- [ ] `SandboxMode::ReadOnly` allows no writes; `WorkspaceWrite` allows writes to `writable_roots`
- [ ] Tests pass: `cargo test -p codex-linux-sandbox`

#### US-034: macOS sandbox via Seatbelt
**Description:** As the system on macOS, I want agent shell commands to run under Apple Seatbelt so that filesystem and network access are restricted.

**Acceptance Criteria:**
- [ ] Seatbelt profile is generated from `seatbelt_base_policy.sbpl` + `seatbelt_network_policy.sbpl` in `codex-rs/sandboxing/src/`
- [ ] `SandboxMode::ReadOnly` generates a read-only Seatbelt profile
- [ ] `SandboxMode::WorkspaceWrite` adds writable paths to the profile
- [ ] `codex sandbox macos -- <cmd>` invokes the Seatbelt sandbox
- [ ] Tests pass: `cargo test -p codex-sandboxing`

#### US-035: Windows sandbox via restricted token
**Description:** As the system on Windows, I want agent shell commands to run under a restricted token with optional private desktop so that they cannot access system resources.

**Acceptance Criteria:**
- [ ] `WindowsSandboxLevel::RestrictedToken` creates a restricted process token
- [ ] `WindowsSandboxLevel::Elevated` allows elevated mode for admin tasks
- [ ] Private desktop isolation is enabled by default (`windows.private_desktop = true`)
- [ ] `codex sandbox windows -- <cmd>` invokes the restricted-token sandbox
- [ ] Tests pass: `cargo test -p codex-sandboxing`

---

### SUBSYSTEM 8 — MCP Integration (`codex-mcp`, `codex-mcp-server`)

#### US-036: Connect to and aggregate MCP servers
**Description:** As the system, I want to connect to all configured MCP servers at session start and aggregate their tools so that the agent can call external capabilities.

**Acceptance Criteria:**
- [ ] `McpConnectionManager` in `codex-rs/codex-mcp/src/mcp_connection_manager.rs` initializes all servers from config
- [ ] Stdio transport: spawns process with `command`, `args`, `env`
- [ ] StreamableHttp transport: connects to `url` with bearer token
- [ ] Tools are discovered and cached with naming pattern `mcp__<server>__<tool>`
- [ ] When tool count exceeds `DIRECT_MCP_TOOL_EXPOSURE_THRESHOLD` (100), deferred exposure via `tool_search` activates
- [ ] Tests pass: `cargo test -p codex-mcp`

#### US-037: Run Codex as an MCP server
**Description:** As an external MCP client (e.g., Claude Desktop), I want to connect to `codex mcp-server` so that Codex's tools are available in other agent frameworks.

**Acceptance Criteria:**
- [ ] `codex mcp-server` starts an MCP protocol server on stdio
- [ ] Codex tools (shell execution, file operations, apply_patch, web_search) are exposed as MCP tools
- [ ] MCP clients can call tools and receive results via the MCP protocol
- [ ] `codex mcp list --verbose` shows all tools from all MCP servers
- [ ] Tests pass: `cargo test -p codex-mcp-server`

---

### SUBSYSTEM 9 — Hooks (`codex-hooks`)

#### US-038: PreToolUse hook — modify or block tool calls
**Description:** As a developer, I want to register a `PreToolUse` hook so that I can inspect, modify, or block tool calls before they execute.

**Acceptance Criteria:**
- [ ] Hook is configured in `codex-rs/hooks/src/` and fires before every tool execution
- [ ] Hook receives: tool name, input JSON, tool description
- [ ] Hook can return `Continue { updated_input }` to modify arguments
- [ ] Hook can return `Blocked(reason)` to prevent execution (reason returned to model)
- [ ] `HookStartedEvent` and `HookCompletedEvent` are emitted
- [ ] Tests pass: `cargo test -p codex-hooks`

#### US-039: PostToolUse hook — inject context after tool execution
**Description:** As a developer, I want to register a `PostToolUse` hook so that I can add context or metadata after a tool executes.

**Acceptance Criteria:**
- [ ] Hook fires after tool execution with: tool_use_id, tool_name, tool_input, tool_response
- [ ] Hook can return `PostToolUseOutcome` with additional context mutations
- [ ] Hook output is injected into the conversation before the next model call
- [ ] Tests pass: `cargo test -p codex-hooks`

#### US-040: SessionStart and UserPromptSubmit hooks
**Description:** As a developer, I want `SessionStart` and `UserPromptSubmit` hooks so that I can inject custom context at session initialization or before each user message.

**Acceptance Criteria:**
- [ ] `SessionStart` hook fires once per new session
- [ ] `UserPromptSubmit` hook fires before each user turn is sent to the model
- [ ] Both hooks can inject additional context or stop execution (`should_stop` flag)
- [ ] Tests pass: `cargo test -p codex-hooks`

---

### SUBSYSTEM 10 — Plugins & Skills (`codex-plugin`, `codex-core-plugins`, `codex-skills`, `codex-core-skills`)

#### US-041: Load and manage plugins
**Description:** As the system, I want to discover and load plugins from the plugin marketplace so that bundled MCP servers, skills, and hooks are automatically available.

**Acceptance Criteria:**
- [ ] `PluginsManager` in `codex-rs/core-plugins/src/manager.rs` loads plugins from `$CODEX_HOME/plugins/`
- [ ] Plugin manifests define: display name, description, MCP server names, app connector IDs, skill presence
- [ ] `PluginCapabilitySummary` is returned per plugin
- [ ] `marketplace.rs` fetches from `openai-curated` and `openai-bundled` marketplaces
- [ ] `codex plugin` CLI command shows installed plugins and marketplace listings
- [ ] Tests pass: `cargo test -p codex-core-plugins`

#### US-042: Skill injection into prompts
**Description:** As the system, I want skills to be injected into the system prompt as tool definitions so that the agent can follow skill-specific instructions for complex tasks.

**Acceptance Criteria:**
- [ ] `SkillMetadata` in `codex-rs/core-skills/src/model.rs` defines: name, description, interface, dependencies, policy
- [ ] `render.rs` converts active skills into tool definitions injected into the model prompt
- [ ] Implicit skills are auto-invoked based on context (`allow_implicit_invocation` policy flag)
- [ ] Skills with MCP server dependencies prompt the user to install missing servers
- [ ] `/skills` slash command lists available skills and toggles them
- [ ] Tests pass: `cargo test -p codex-core-skills`

---

### SUBSYSTEM 11 — App-Server (`codex-app-server`, `codex-app-server-protocol`)

#### US-043: Start the app-server and accept JSON-RPC connections
**Description:** As a client (VS Code extension, TUI, desktop app), I want to connect to the app-server over stdio or UDS so that I can manage Codex sessions through a typed RPC API.

**Acceptance Criteria:**
- [ ] App-server starts with `--listen stdio://` (newline-delimited JSON) or `--listen unix://` (WebSocket over UDS)
- [ ] `MessageProcessor` in `codex-rs/app-server/src/` routes all v2 RPC methods to the correct handler module
- [ ] `ConnectionSessionState` tracks per-connection auth state and capabilities
- [ ] `OutgoingMessageSender` routes responses and notifications back to the correct client
- [ ] Overloaded connections return error code `-32001`
- [ ] Tests pass: `cargo test -p codex-app-server`

#### US-044: Thread lifecycle via app-server v2 protocol
**Description:** As a client, I want to start, resume, fork, archive, and list threads via the app-server v2 RPC so that I can manage multiple conversations programmatically.

**Acceptance Criteria:**
- [ ] `thread/start` creates a new thread and returns thread ID
- [ ] `thread/resume` resumes a previous thread by ID
- [ ] `thread/fork` creates a branch from an existing thread
- [ ] `thread/archive` / `thread/unarchive` toggles archive state
- [ ] `thread/list` returns paginated list with `cursor`, `limit`, `data`, `next_cursor`
- [ ] `thread/read` returns full thread details
- [ ] `thread/setName` updates the display name
- [ ] `thread/goal/set`, `thread/goal/get`, `thread/goal/clear` manage thread goals
- [ ] `thread/memoryMode/set` sets the memory mode
- [ ] Tests pass: `cargo test -p codex-app-server`

#### US-045: Turn execution via app-server v2 protocol
**Description:** As a client, I want to start, interrupt, and steer turns via RPC so that I can drive agent execution and handle mid-turn interventions.

**Acceptance Criteria:**
- [ ] `turn/start` submits a user message and begins agent execution
- [ ] `turn/interrupt` cancels the current turn
- [ ] `turn/steer` injects a mid-turn user correction
- [ ] Notifications `turn/started`, `turn/completed`, `item/started`, `item/completed` are sent for each event
- [ ] `item/agentMessage/delta` notifications stream text incrementally
- [ ] `item/reasoning/delta` notifications stream reasoning tokens
- [ ] Tests pass: `cargo test -p codex-app-server`

#### US-046: Command execution via app-server
**Description:** As a client, I want to run interactive terminal commands via the app-server `command/exec` RPC so that I can embed a terminal emulator in the client.

**Acceptance Criteria:**
- [ ] `command/exec` starts a process and returns a command ID
- [ ] `command/exec/write` sends stdin data to the running process
- [ ] `command/exec/terminate` sends a termination signal
- [ ] `command/exec/resize` resizes the terminal pty
- [ ] `command/exec/outputDelta` notifications stream stdout/stderr
- [ ] `command/exec/exited` notification includes exit code
- [ ] Tests pass: `cargo test -p codex-app-server`

#### US-047: Configuration management via app-server
**Description:** As a client, I want to read and write configuration via the app-server `config/*` RPC so that the client UI can provide settings management.

**Acceptance Criteria:**
- [ ] `config/read` returns current config as structured JSON
- [ ] `config/write` accepts partial config updates and persists them
- [ ] `config/import` imports a config file
- [ ] `config/detect` discovers config files on disk
- [ ] Config fields use snake_case on wire (mirrors config.toml keys)
- [ ] Tests pass: `cargo test -p codex-app-server`

---

### SUBSYSTEM 12 — Exec-Server (`codex-exec-server`)

#### US-048: Start exec-server and accept connections
**Description:** As an operator, I want to run `codex exec-server` as a standalone daemon so that remote or isolated environments can execute commands on behalf of the agent.

**Acceptance Criteria:**
- [ ] `codex exec-server --listen ws://0.0.0.0:PORT` starts a WebSocket JSON-RPC listener
- [ ] `codex exec-server --listen stdio` starts on stdio
- [ ] `codex exec-server --remote <URL> --executor-id <ID>` registers with a central executor registry
- [ ] `FileSystemSandboxContext` restricts file operations to allowed paths
- [ ] Tests pass: `cargo test -p codex-exec-server`

#### US-049: Remote command execution via exec-server protocol
**Description:** As a client, I want to execute commands and perform file operations on the exec-server so that the agent can work in remote or sandboxed environments.

**Acceptance Criteria:**
- [ ] `initialize` → `exec` → streaming `exec/outputDelta` → `exec/exited` lifecycle works
- [ ] `exec/write` sends stdin to running process
- [ ] `exec/terminate` / `exec/close` terminates process
- [ ] `fs/readFile`, `fs/writeFile`, `fs/createDirectory`, `fs/readDirectory`, `fs/getMetadata`, `fs/copy`, `fs/remove` all work
- [ ] `http/request` with streaming `httpRequest/bodyDelta` works
- [ ] Tests pass: `cargo test -p codex-exec-server`

---

### SUBSYSTEM 13 — Desktop App Bridge (`codex-cli/desktop_app`)

#### US-050: Launch or install the Codex desktop app
**Description:** As a user, I want `codex app [PATH]` to open a workspace in the Codex desktop app, installing it first if needed.

**Acceptance Criteria:**
- [ ] `run_app_open_or_install()` in `codex-rs/cli/src/desktop_app/mod.rs` opens workspace in installed app
- [ ] If app is not installed, installer is downloaded and run (macOS: `mac.rs`, Windows: `windows.rs`)
- [ ] `--download-url <URL>` overrides the default installer download location
- [ ] On macOS: Codex.app is launched via native macOS open APIs
- [ ] On Windows: Codex installer runs silently then launches the app
- [ ] Tests pass: `cargo test -p codex-cli`

---

### SUBSYSTEM 14 — IPC / Transport Layer (`codex-uds`, `codex-stdio-to-uds`)

#### US-051: Unix domain socket transport
**Description:** As the system, I want a cross-platform UDS transport so that app-server, TUI, and desktop app can communicate securely via Unix sockets on Linux, macOS, and Windows.

**Acceptance Criteria:**
- [ ] `codex-rs/uds/src/lib.rs` exposes `UnixListener` and `UnixStream` with async read/write
- [ ] `prepare_private_socket_directory()` creates the socket dir with `0o700` permissions
- [ ] `is_stale_socket_path()` detects and removes stale sockets
- [ ] Works on Windows via `uds_windows` crate
- [ ] Tests pass: `cargo test -p codex-uds`

#### US-052: Stdio-to-UDS bridge
**Description:** As an external process, I want `codex stdio-to-uds <SOCKET_PATH>` to relay stdin/stdout to a UDS socket so that stdio-only clients (e.g., VS Code) can speak to the app-server.

**Acceptance Criteria:**
- [ ] `codex-rs/stdio-to-uds/src/lib.rs` performs bidirectional async copy between stdio and UDS
- [ ] Shutdown propagates correctly when either end closes
- [ ] Tests pass: `cargo test -p codex-stdio-to-uds`

---

## Functional Requirements

- **FR-01:** The system must accept `Op` variants from clients and route them to the correct handler in `submission_loop()` (`codex-rs/core/src/session/handlers.rs`)
- **FR-02:** All shell command executions must pass through `process_exec_tool_call()` and be subjected to the active `SandboxMode` restrictions
- **FR-03:** When `AskForApproval` policy requires it, an `ExecApprovalRequestEvent` must be emitted and the agent loop must block until `Op::ExecApproval` is received
- **FR-04:** All agent-to-client events must conform to the `EventMsg` enum in `codex-rs/protocol/src/protocol.rs`
- **FR-05:** The TUI must render incrementally — new transcript entries must not cause full re-renders
- **FR-06:** The app-server v2 protocol must be strictly camelCase on the wire (`#[serde(rename_all = "camelCase")]`) except config endpoints (snake_case)
- **FR-07:** All new v2 protocol types must have `#[ts(export_to = "v2/")]` for TypeScript generation
- **FR-08:** MCP server connections must survive transient failures with graceful error reporting (not panics)
- **FR-09:** Sandbox restrictions must be enforced at the OS level (not just policy checks) for `read-only` and `workspace-write` modes
- **FR-10:** Session resume must detect cwd changes and prompt the user to resolve them before resuming
- **FR-11:** Context compaction must fire automatically when token count exceeds `model_auto_compact_token_limit`
- **FR-12:** Plugin marketplace sync must happen asynchronously and not block session startup
- **FR-13:** Hook execution must emit `HookStartedEvent` and `HookCompletedEvent` for every hook invocation
- **FR-14:** The exec-server must support both local (`--listen ws://`) and remote-registered (`--remote`) deployment modes
- **FR-15:** All public crate APIs must be private-first with explicit `pub use` exports

---

## Non-Goals (Out of Scope)

- **No cloud execution** — Codex runs entirely on the user's machine; cloud-based Codex Web is a separate product
- **No built-in code editor** — Codex does not embed a code editor; it uses `file_opener` config to launch external editors
- **No GUI framework** — The desktop app itself (Electron/Swift/etc.) is not in this repository; only the bridge `codex app` command is
- **No automatic priority assignment** — Exec policy rules are manually defined; no ML-based policy inference
- **No general documentation site** — `docs/` folder is for dev workflow docs only; product docs are external
- **No workspace-level multi-user real-time collaboration** — collaboration mode is experimental and single-user per session
- **No plugin authoring tools** — PRD covers plugin consumption, not the toolchain for building new plugins
- **No v1 app-server API additions** — all active API development is in v2 only

---

## Technical Considerations

### Crate Architecture
- All Rust crates are in `codex-rs/` Cargo workspace; crate names are prefixed `codex-`
- Resist adding to `codex-core` — it is already large; prefer new crates or existing smaller ones
- Target module files under 500 LoC; refactor into sub-modules when approaching 800 LoC
- Build system: Cargo (development), Bazel (CI/release)

### Async Runtime
- All async code uses Tokio
- Async traits use native RPITIT: `fn foo(&self) -> impl Future<Output = T> + Send;` — no `#[async_trait]`
- Avoid holding `tokio::sync::MutexGuard` / `RwLockGuard` across await points

### Protocol / Serialization
- `codex-protocol` uses serde with `#[serde(rename_all = "camelCase")]` for v2
- Config endpoints use snake_case to mirror `config.toml` keys
- Timestamps: `i64` Unix seconds, named `*_at`
- Pagination: `cursor: Option<String>`, `limit: Option<u32>`, `data: Vec<T>`, `next_cursor: Option<String>`

### TUI
- ratatui with Stylize helpers; no hardcoded `.white()` or `.black()`
- Text wrapping via `textwrap::wrap` for plain strings; `word_wrap_lines()` for ratatui `Line`s
- Snapshot tests via `insta`; any UI change must include snapshot update

### Testing
- Unit tests: `cargo test -p codex-<crate>`
- Full suite: `just test` (requires `cargo-nextest`)
- Snapshot tests: `cargo insta accept -p codex-tui` after review
- Use `pretty_assertions::assert_eq` in all test modules
- Use `codex_utils_cargo_bin::cargo_bin()` to locate binaries in tests

### Dependency Management
- After any `Cargo.toml` / `Cargo.lock` change: run `just bazel-lock-update` then `just bazel-lock-check`
- After `ConfigToml` changes: run `just write-config-schema`
- After app-server protocol changes: run `just write-app-server-schema` + `cargo test -p codex-app-server-protocol`

---

## Success Metrics

- A user can start `codex` with no arguments and have a working chat session within 5 seconds on a warmed machine
- Non-interactive `codex exec "prompt"` completes and exits correctly with code 0 on success
- All shell commands execute inside OS-level sandboxes (verified by `codex sandbox <os>` test utility)
- TUI renders at 60fps with no full re-renders on incremental transcript updates
- App-server JSON-RPC round-trip latency (local UDS) is under 5ms for non-agent calls
- `cargo test -p codex-core` passes with zero failures on Linux, macOS, and Windows
- Context compaction fires automatically and resumes the conversation without user intervention

---

## Open Questions

1. **Realtime voice** — Is the realtime voice feature (`/realtime`) production-ready or permanently experimental? What is the graduation criteria?
2. **Multi-agent collaboration** — What is the intended scope of the collaboration mode (`/collab`)? When will it exit experimental status?
3. **Plugin marketplace** — Which plugins are in the curated allowlist (`github`, `notion`, `slack`, `gmail`)? Is the list static or configurable?
4. **Exec-server remote registry** — Is there a public registry endpoint for `codex exec-server --remote`? Is this feature available to all users?
5. **Windows sandbox** — `WindowsSandboxLevel::RestrictedToken` is not the default (`disabled`). What is the migration plan for enabling sandboxing on Windows by default?
6. **App-server v1 deprecation** — When will v1 be removed? Are there any active clients still on v1?
7. **`codex review` command** — What is the expected behavior of non-interactive review mode? Is it considered stable?
8. **Token count telemetry** — `TokenCountEvent` is emitted — what is the exact token counting methodology (input tokens, output tokens, cached tokens)?
