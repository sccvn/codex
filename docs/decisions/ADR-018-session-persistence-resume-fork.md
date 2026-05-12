# ADR-018: Session Persistence via Local History for Resume and Fork

## Status
Accepted

## Date
2026-05-12

## Context
Long coding sessions take hours. Users need to:
- Resume a session after closing the terminal
- Fork a session to explore an alternative approach without losing the original
- Pick up a session on a different machine (with shared storage)
- Roll back to an earlier point in a conversation

The OpenAI Responses API does maintain server-side state (via `previous_response_id`), but this state is opaque: the client cannot inspect it, it can expire, and it is not portable.

## Decision
Persist session history locally as a JSONL (newline-delimited JSON) log in `$CODEX_HOME/history/`. Each turn's items are stored locally. On resume:
1. Local history is loaded and replayed to reconstruct `previous_response_id` chains
2. If the session's cwd has changed, the user is prompted to choose the correct working directory

Fork creates a new session that copies local history up to the fork point; the new session has its own `session_id` and history file.

`ThreadManager` in `codex-rs/core/src/session/` manages the in-memory session state; `--resume-last`, `--resume-session-id`, and `--resume-picker` CLI flags control session selection at startup.

## Alternatives Considered

### Server-side history only (rely on Responses API)
- Pros: No local storage management
- Cons: History can expire; not accessible offline; cannot fork or inspect history without API calls; not portable across devices
- Rejected: Local history ownership is important for a privacy-conscious, locally-running tool

### SQLite for history storage
- Pros: Queryable, transactional, compact
- Cons: Requires schema migrations; more complex than append-only JSONL; harder to inspect manually
- Rejected: Append-only JSONL is sufficient for the current use case; can migrate to SQLite later if querying needs arise

### Git-based history
- Pros: Built-in branching (fork = branch), history inspection via `git log`
- Cons: Heavy dependency; users may not want Codex history in their git repos; fork semantics don't map cleanly to git branches
- Rejected: JSONL with explicit fork IDs is simpler

## Consequences
- History is stored in `$CODEX_HOME/history/<session_id>.jsonl`
- `--resume-picker` renders a TUI list of recent sessions with cwd, timestamp, and summary
- `Op::ThreadRollback { num_turns }` drops the last N turns from in-memory state (does not rewrite history files)
- Context compaction (`Op::Compact`) produces a summary that replaces the detailed history for the compacted portion
