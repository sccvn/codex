# PRD: Codex Platform — Product Overview for Stakeholders

> **Audience:** Product stakeholders, CPO, roadmap planning, executive review
> **Scope:** As-is documentation — all surfaces in production today (CLI, TUI, MCP server, app-server, exec-server, desktop bridge)
> **Depth:** High-level epics grouped by feature area
> **Date:** 2025-05-12

---

## Introduction

**Codex** is OpenAI's open-source, locally-running coding agent. It accepts natural-language instructions and autonomously executes multi-step software engineering tasks: writing and editing code, running shell commands, applying patches, searching the web, and calling external tools — all while respecting configurable safety boundaries enforced at the OS level.

Codex is deployed as a Rust-native binary distributed via npm, with multiple interaction surfaces serving developers in different contexts: an interactive terminal UI, a scriptable CLI for CI/CD, a JSON-RPC daemon powering desktop and VS Code integrations, an MCP server for composability with other AI agents, and a sandboxed remote execution service.

The agent loop is powered by OpenAI's Responses API. Users authenticate via ChatGPT OAuth or API key. Every shell command the agent runs is confined by a platform-native sandbox (Linux, macOS, Windows) so the agent cannot exceed the permissions the user has granted.

---

## Goals

- Provide an autonomous coding agent that runs entirely on the user's machine with no cloud dependency beyond the OpenAI API
- Allow the model to read/write files, run shell commands, and call external tools within configurable safety guardrails
- Support multiple interaction surfaces (TUI, CLI, desktop app, VS Code, mobile) through a unified backend protocol
- Enable extensibility via MCP servers, hooks, plugins, and skills without modifying core code
- Enforce platform-appropriate OS-level sandboxing on Linux, macOS, and Windows
- Support session persistence: users can resume, fork, and summarize previous conversations
- Support multi-agent workflows: spawn sub-agents, orchestrate parallel tasks, monitor progress

---

## Feature Areas (Epics)

---

### EPIC 1 — Interactive Terminal UI (TUI)

**What it is:** The primary human-facing workspace for real-time collaboration with the coding agent.

The Codex TUI is a full-screen terminal application where users chat naturally with an AI coding agent, approve or deny sensitive operations, monitor command execution in real time, and manage complex multi-threaded agent workflows. It is purpose-built for developers who want a powerful, keyboard-driven workspace — combining the feel of a chat interface with the control of a terminal.

**Capabilities:**

**Core Chat & Interaction**
- Full-screen chat interface: transcript view + message composer rendered at up to 60 fps
- Streaming agent responses render in real time; reasoning tokens displayed separately for models that expose reasoning steps
- Natural-language input; supports markdown, inline code, and file path references
- Voice input (`/realtime`): live audio conversations with the agent, configurable input/output devices

**Approval & Safety Flows**
- **Command approval modal**: before running shell commands (when policy requires it), user sees command text, working directory, and accepts/denies; approved decisions persist as policy rules
- **Patch approval modal**: user reviews and approves/denies file edits with diff view before they are applied
- **Network approval modal**: user reviews and approves external HTTP requests before they execute
- "Approve for session" or "learn this pattern" options reduce repetitive prompts for trusted operations

**Session Management**
- Resume picker: interactive list of past sessions with CWD, date, and summary; filter by directory
- Fork: create a new conversation branching from any past session to explore alternatives
- Conversation compaction (`/compact`): summarize long transcripts to stay within model context limits; with auto-warning when limit is near
- Session naming: custom titles for tracking parallel workstreams

**Model & Reasoning Control**
- Model picker (`/model`): switch active model mid-session from a browsable catalog
- Reasoning effort control (`Alt+,` / `Alt+.`): tune depth of agent reasoning (speed vs. quality trade-off)
- Live status bar: current model, reasoning effort, approval policy, and token usage

**Multi-Agent Workflows**
- Agent thread navigator (`Alt+←` / `Alt+→`): cycle between active agent threads when agent spawns sub-agents
- Agent picker (`/agent`): see all active threads with name, role, and status (running/idle/complete)
- Sub-agent completion notifications surface in parent thread

**Slash Commands (50+ available)**
- Model & session: `/model`, `/compact`, `/resume`, `/fork`, `/new`, `/rename`
- Safety & review: `/approve`, `/review`, `/diff`, `/permissions`
- Configuration & UX: `/vim` (Vim keybindings), `/keymap`, `/theme`, `/statusline`, `/title`
- Tools & extensibility: `/mcp`, `/skills`, `/hooks`, `/plugins`, `/memories`
- Utilities: `/status`, `/copy`, `/logout`, `/quit`

**Onboarding Experience**
- Guided first-launch flow: Welcome → Authentication → Trust Directory
- ChatGPT OAuth auto-opens browser for device code; API key as fallback
- Trust prompt for read/write access to working directory; opt-in for crash reporting

**Keyboard & UX Customization**
- Full Vim mode (hjkl navigation, dd, yy, gg/G); toggled via `/vim`
- Terminal title and status line customization via slash commands
- `--no-alt-screen` mode for Zellij/tmux compatibility (preserves terminal scrollback)
- Syntax highlighting with theme selector; side-by-side diff rendering with add/remove colors

---

### EPIC 2 — CLI: Non-Interactive & Scripted Execution

**What it is:** A single-command interface for scripting, CI/CD automation, and power users who want agent execution without a UI.

The Codex CLI provides everything the TUI does, but headless. `codex exec` runs a task, returns output, and exits with a success or failure code — making it composable with shell scripts, GitHub Actions, Make targets, and any other automation workflow. All safety controls (sandbox mode, approval policy) are configurable as flags.

**Capabilities:**

**Interactive mode** (`codex [PROMPT]`)
- Launches the full TUI; optional prompt pre-fills the first message
- Same session management, model selection, and safety controls as TUI

**Non-interactive / scripted mode** (`codex exec PROMPT`)
- Single-shot execution: agent accepts the task, runs tools, produces output, exits
- `--json` flag: emit all agent events as JSONL (structured log ingestion, pipeline parsing)
- `--output-last-message <file>`: capture the agent's final response to a file
- `--output-schema <file>`: constrain response to a JSON schema (structured output)
- `--ephemeral`: disable session persistence (useful for stateless CI runs)

**Safety & sandbox flags**
- `--sandbox <mode>`: read-only / workspace-write / danger-full-access per invocation
- `--ask-for-approval <policy>`: never / on-request / on-failure / granular
- `--dangerously-bypass-approvals-and-sandbox`: removes all safety checks (advanced use only)
- `--ignore-rules`: skip execpolicy rule enforcement

**Model & capability flags**
- `--model <name>`: select model at invocation (e.g., `gpt-4o`, `o1`)
- `--search`: enable live web search within agent responses
- `--full-auto`: run fully autonomously without pausing for approvals

**Authentication management**
- `codex login` / `codex logout`: manage ChatGPT OAuth and API key credentials
- `codex login --switch`: switch between stored accounts

**Session management**
- `codex resume [--last | --picker | <SESSION_ID>]`: resume past conversations
- `codex fork [--last | --picker | <SESSION_ID>]`: branch from past session

**MCP tool management**
- `codex mcp list / add / remove / login / logout`: manage external MCP tool servers

**Other utilities**
- `codex completion <shell>`: generate bash/zsh shell completions
- `codex update`: upgrade to the latest Codex release
- `codex app`: open Codex desktop application
- `codex sandbox <os> -- <cmd>`: test sandbox behavior on a specific platform
- `codex mcp-server`: run Codex as an MCP tool provider (see Epic 4)

---

### EPIC 3 — Sandboxing & Safety

**What it is:** OS-level isolation that enforces the user's permission profile, preventing the agent from exceeding what it has been granted — regardless of model behavior.

Codex treats sandboxing as a first-class product feature, not an afterthought. Every shell command the agent runs passes through a platform-native isolation layer. This is not policy-checking in application code — it is enforced by the operating system kernel.

**Three Sandbox Modes**

| Mode | What the agent can do | Best for |
|---|---|---|
| **read-only** | Read any file; no writes, no network | Code analysis, audits, documentation generation |
| **workspace-write** | Read any file; write only to configured directories | Active development (recommended default) |
| **danger-full-access** | Unrestricted | Admin tasks, trusted automation |

**Platform Protection**

- **Linux**: Bubblewrap (user namespace isolation) + Landlock (kernel-level filesystem rules) + Seccomp (syscall allowlist). Defense-in-depth — multiple independent layers.
- **macOS**: Apple Seatbelt (`sandbox-exec`). Dynamically generated profiles restrict file access, IPC, network, and device access. Enforced by the macOS security model.
- **Windows**: Restricted process token. Revoked admin and file-system write privileges. Optional private desktop isolation. (Note: Windows protection is less mature than Linux/macOS; labeled experimental.)

**Approval Policies**

| Policy | Behavior |
|---|---|
| **Never** | Commands run immediately; failures returned to model |
| **On-Request** | Agent requests approval for risky operations (default for TUI) |
| **On-Failure** | Sandbox-contained failures prompt re-run approval |
| **Granular** | Per-category control: shell commands, MCP tools, sandbox escapes, skills |

**Approval routing options**: User review (default), AI auto-review (for high-volume workflows), or legacy guardian subagent.

**Network Access Control**
- Network is **off by default** in read-only and workspace-write modes
- Can be enabled per-profile with domain allowlisting and SOCKS5/HTTP proxy support

**Named Permission Profiles**
- Users define multiple named profiles in config (e.g., `dev`, `review-only`, `unrestricted`)
- Each profile specifies filesystem rules (read-only paths, writable roots) and network rules
- Switched live via `/permissions` slash command mid-session

---

### EPIC 4 — MCP Server: Codex as a Composable Tool

**What it is:** Codex exposes itself as a standards-compliant Model Context Protocol (MCP) tool provider — turning Codex into a reusable capability that any MCP-compatible AI agent can call.

Running `codex mcp-server` starts a lightweight JSON-RPC service over stdio. External agents (e.g., Claude Desktop, custom AI applications) connect to it and invoke Codex tools — shell execution, file patching, web search, MCP proxying — as named callable functions. Each tool call spawns an isolated Codex thread, executes the work, and returns structured results.

**Key Capabilities**
- **Tool advertisement**: Exposes all Codex tools plus any installed skill tools with JSON schemas and descriptions
- **Sandboxed execution**: Each tool call runs in an isolated Codex thread with caller-specified working directory and permission profile
- **Streaming results**: Returns output in real time including progress notifications and structured completion data
- **Approval surfacing**: Dangerous operations (shell commands, file patches) surface to the caller for approval before execution
- **OAuth pass-through**: MCP server OAuth flows for integrated services are handled transparently

**Transport**: stdio (JSONL) — portable across all platforms; HTTP/WebSocket transport planned.

**Use Cases**
- Claude Desktop or custom AI apps invoke Codex for code review, refactoring, or file operations without a separate Codex UI
- Compose multi-agent pipelines where Codex is the "code execution layer" alongside specialized tools
- Enterprise integrations embed Codex as a sandboxed execution microservice

---

### EPIC 5 — App-Server: The Backend for Rich Interfaces

**What it is:** A persistent JSON-RPC 2.0 daemon that powers VS Code extensions, the Codex desktop app, and mobile apps — providing a full conversation-and-execution backend over a bidirectional streaming protocol.

Unlike a stateless API, the app-server maintains full session state, streams real-time events to connected clients, and handles the full Codex agent loop including sandboxing, tool routing, MCP servers, and session persistence. It is the unified control plane for all non-TUI Codex surfaces.

**Core Abstractions**
- **Thread**: A persisted conversation session on disk; resumable, forkable, archivable
- **Turn**: One user message + full agent response, streamed in real time
- **Item**: Individual units of conversation (user text, agent reasoning, command output, file edits) — persisted and reused as context

**Conversation & Session Management**
- `thread/start`, `thread/resume`, `thread/fork`, `thread/list`, `thread/archive`
- `thread/goal/set` — attach goals and token budgets for long-running tasks
- `thread/compact/start` — auto-summarize long conversations to prevent context overflow

**Real-Time Agent Interaction**
- `turn/start` — submit user input; immediately returns turn ID then streams progress
- `turn/interrupt` — cancel an in-flight turn
- `turn/steer` — inject course-correction mid-execution without starting a new turn
- Streaming events: `item/agentMessage/delta`, `item/commandExecution/outputDelta`, `item/completed`, approval requests

**Tools & Environment**
- `command/exec` — run ad-hoc sandboxed commands outside a turn
- `skills/list` — discover local and marketplace skills
- `plugin/list`, `plugin/install` — browse and install extensions
- `environment/add` — register remote exec-servers for distributed execution

**Configuration Management**
- `config/read` — get effective merged configuration
- `config/value/write`, `config/batchWrite` — update config on disk with live hot-reload

**Transport Modes**

| Transport | Use Case |
|---|---|
| **Stdio (JSONL)** | CLI tools, `codex exec`, scripting |
| **Unix Domain Socket (WebSocket)** | VS Code extension, desktop app (same machine) |
| **Remote Control (cloud-bridged)** | Mobile app, remote workstation control |

Socket protected by Unix file permissions (owner-only, mode 0700).

**Protocol Versions**: v2 (current, JSON-RPC 2.0 bidirectional, all new clients); v1 (legacy, maintained for backward compatibility).

---

### EPIC 6 — Exec-Server: Isolated Remote Execution

**What it is:** A lightweight WebSocket daemon that decouples subprocess spawning and filesystem operations from the main agent loop, enabling distributed, containerized, and fault-isolated execution.

**Use Cases**
- Run agent commands on a remote machine, container, or Kubernetes pod
- Fault isolation: agent hangs or crashes do not affect the conversational loop
- Multi-tenant safety: multiple Codex threads share one exec-server with per-connection sandbox policies

**Process Management**
- `process/start`, `process/write`, `process/read`, `process/terminate`
- Real-time streaming: stdout/stderr chunks emitted as sequenced notifications
- Supports PTY (pseudo-terminal) for interactive programs

**Filesystem Operations (Sandbox-Aware)**
- `fs/readFile`, `fs/writeFile`, `fs/createDirectory`, `fs/remove`, `fs/copy`, `fs/readDirectory`
- Each request accepts a sandbox policy (`ReadOnly`, `WorkspaceWrite`) enforced by the OS
- `WorkspaceWrite` denies writes outside the configured workspace root

**Transport**: WebSocket (default `ws://127.0.0.1:3000`); supports remote registration via rendezvous WebSocket with bearer token auth.

---

### EPIC 7 — Desktop Bridge & Remote Control

**What it is:** The IPC layer that connects native desktop applications (macOS, Windows, Linux) and mobile clients to the local Codex daemon, enabling seamless GUI experiences without sacrificing local execution security.

**Core Bridge: `stdio-to-uds`**
- Proxies the full WebSocket protocol between any process (Electron app, native app) and the app-server's Unix Domain Socket
- Allows desktop apps to communicate with app-server without any knowledge of Unix socket internals
- Socket protected at the OS level (mode 0700; only the owning user can connect)

**Daemon Lifecycle Management**
- `codex app-server daemon start / stop / restart` — manage app-server as a background process
- `codex app-server daemon enable-remote-control` — enroll workstation for mobile/cloud control
- `codex app-server daemon bootstrap` — one-command setup: start daemon + enable remote control + start auto-updater
- Pidfile-backed; survives terminal sessions; auto-updates binary in background

**Desktop Integration (All Platforms)**
- macOS/Windows: Codex.app detects and auto-starts local daemon; connects via stdio-to-uds bridge
- Linux: direct Unix socket connection (no bridge needed)
- Multi-app coexistence: multiple desktop windows share one daemon and see consistent thread state

**Remote Control (Mobile → Desktop)**
1. User runs `codex app-server daemon bootstrap --remote-control` on workstation
2. Daemon registers with OpenAI backend and opens rendezvous WebSocket
3. Mobile app (iOS/Android) lists enrolled devices; user selects workstation
4. Backend proxies JSON-RPC requests to workstation; all execution stays local
5. Mobile is a thin client; no code or data leaves the workstation

---

### EPIC 8 — Authentication & Authorization

**What it is:** Secure, flexible credential management supporting both consumer (ChatGPT) and developer (API key) workflows, with OS-native token storage and automatic refresh.

**Authentication Methods**

| Method | User Experience | Best For |
|---|---|---|
| **ChatGPT OAuth** (device code + PKCE) | First-launch wizard → browser redirect → one-time code → done. Token auto-refreshes. | ChatGPT Plus/Team subscribers; interactive users |
| **API Key** | Paste key once at `codex login --api-key`. Simple but requires manual rotation. | Developers; CI/CD systems; service accounts |

**Token Storage Options**

| Mode | Where credentials live | Best For |
|---|---|---|
| **keyring** (default) | OS native: macOS Keychain, Windows Credential Manager, Linux Secret Service | Interactive users on personal machines |
| **file** | `$CODEX_HOME/auth.json` (owner-readable only) | Shared machines, portability |
| **ephemeral** | Memory only; lost on exit | CI/CD, containers, untrusted environments |
| **auto** | Try keyring; fall back to file | Default; works everywhere |

**Token Lifecycle**
- OAuth tokens auto-refresh within an 8-hour window; user never sees auth prompts during normal work
- API key failures surface a re-authentication prompt immediately
- Multi-account support: `codex login --switch` to change active credentials

**MCP Server Auth**
- MCP servers can authenticate via OAuth (scopes stored in same credential store) or environment variable bearer tokens

---

### EPIC 9 — Configuration & Customization

**What it is:** A layered configuration system that lets users and teams control every aspect of Codex behavior — model selection, sandbox mode, approval policy, tool integrations, and UI preferences — at global or project level.

**Configuration Layers** (highest priority wins)
1. CLI flags
2. Workspace config (`.codex/config.toml` in project root)
3. Global config (`~/.config/codex/config.toml`)

**Key Configurable Areas**

| Area | What Users Control |
|---|---|
| **Model & Reasoning** | Model name, provider, reasoning effort, reasoning token display, custom system instructions, model catalog |
| **Sandbox & Safety** | Sandbox mode, writable roots, network access per mode, approval policy, approval reviewer |
| **Permissions** | Named permission profiles (filesystem + network rules per profile) |
| **Shell Environment** | Which environment variables the agent inherits |
| **MCP Servers** | Per-server command, args, env, OAuth, tool whitelist/blacklist, approval mode, startup timeout |
| **Plugins** | Enable/disable plugins and per-plugin MCP server + tool overrides |
| **Skills** | Enable/disable bundled skill groups; add custom skill paths; control auto-injection |
| **Hooks** | Shell scripts triggered on agent lifecycle events |
| **UI & Interaction** | Alt-screen mode, keymap, file opener (external editor), session history settings, terminal detection |
| **Windows-specific** | Sandbox level, private desktop isolation |

---

### EPIC 10 — Extensibility: Hooks, Plugins & Skills

**What it is:** Three complementary extension mechanisms that let users and third parties customize agent behavior and integrate external tools without touching Codex internals.

#### Hooks: Event-Driven Automation
Shell scripts that execute in response to agent lifecycle events — enabling custom monitoring, logging, notifications, CI/CD integration, and guardrails.

**Supported events**: `PreToolUse`, `PostToolUse`, `PermissionRequest`, `PreCompact`, `PostCompact`, `PreTurn`, `PostTurn`

**Example uses**:
- Notify Slack when the agent runs a deployment command
- Log all file edits to an audit trail
- Block specific shell patterns (e.g., `rm -rf`) before they execute
- Trigger a test suite after `PostToolUse` on a patch

**Configuration**: Defined in `config.toml` with matcher syntax (conditional on tool type, command content, etc.); can run synchronously or asynchronously.

#### Plugins: Bundled Extensions
Third-party packages (Claude Code plugins) that contribute MCP servers, hooks, and skills in one installable unit — no manual per-server setup.

**User control**: Enable/disable at plugin level or per-server/tool level; set approval policies per plugin.

**Example plugin**: A GitHub plugin might provide a GitHub MCP server (read repos, create PRs, check issues), a Slack MCP server, and a "Create PR" skill — installed as one unit.

#### Skills: Reusable Agent Instructions
Markdown files that teach the agent to perform specific tasks. No compilation required — skills are portable instructions.

**Bundled skills** (built-in): web search, code review, git workflow, test-driven development, debugging, and more.
**Custom skills**: user-defined markdown files in `~/.codex/skills/` or configured paths.

**Discovery**: Active skills are surfaced to the agent automatically; no need to re-prompt the same instructions every session.

#### MCP Integration
The protocol layer connecting Codex to external tool servers. Any JSON-RPC service that implements MCP becomes an agent tool.

**Transports**: Stdio (spawn local process), Streamable HTTP (remote endpoint with bearer token), WebSocket (experimental).

**Example MCP servers**: GitHub, Slack, Jira, Confluence, database clients, custom enterprise tools.

**Tool policy controls**: Whitelist/blacklist tools per server; set approval mode (always/on-request/never) per server.

---

## Non-Goals (Out of Scope Today)

- **Cloud-hosted execution**: All agent execution runs locally on the user's machine; no server-side code execution service
- **Collaborative real-time editing**: Multiple humans editing the same session simultaneously is not supported
- **IDE-native UI**: The TUI and app-server power IDE integrations, but a fully-native IDE panel is a client-side concern
- **Model fine-tuning or training**: Codex uses the OpenAI Responses API as-is; no in-product fine-tuning
- **Codex-hosted authentication**: Auth delegates to OpenAI/ChatGPT; no first-party identity provider
- **Windows secure-by-default sandboxing**: Windows sandbox is experimental; Linux and macOS have production-grade sandboxing

---

## Technical Considerations

- **Language & distribution**: Core agent written in Rust (~90 crates); distributed via npm wrapper for broad reach
- **Model dependency**: Powered by OpenAI Responses API; model selection limited to what the API exposes
- **Platform maturity**: Linux sandboxing (bubblewrap + landlock + seccomp) is the most mature; macOS Seatbelt is production-ready; Windows is experimental
- **Session storage**: Conversations stored on-device as JSONL + SQLite metadata in `$CODEX_HOME`; no cloud backup by default
- **Protocol versioning**: App-server v2 is the active protocol; v1 maintained for compatibility only

---

## Success Metrics

- Developers complete coding tasks (write feature, fix bug, refactor) end-to-end without leaving the TUI
- `codex exec` integration in CI/CD pipelines: tasks complete without human intervention
- Zero unintended file or system modifications outside the configured sandbox scope
- VS Code extension and desktop app connect to app-server with under 1 second startup latency
- External AI agents (Claude Desktop, custom) successfully invoke Codex via MCP with tool round-trip under 5 seconds
- First-time user completes authentication and submits first task in under 3 minutes

---

## Open Questions

1. **Windows sandboxing roadmap**: When does Windows reach parity with Linux/macOS protection? Is it gating enterprise adoption?
2. **Cloud execution surface**: Is there a roadmap for server-side execution (no local install required)?
3. **Mobile app status**: The remote control protocol exists — is an iOS/Android client in active development?
4. **Marketplace for plugins/skills**: Is there a curated plugin marketplace, or is discovery ad-hoc today?
5. **Multi-user / team sessions**: Is collaborative use (multiple developers sharing a session or session history) on the roadmap?
6. **Pricing / model access tiers**: How does API key vs ChatGPT OAuth differ in terms of model availability and rate limits?
7. **Telemetry and analytics**: What usage data is collected today, and what instrumentation exists for understanding user workflows?
