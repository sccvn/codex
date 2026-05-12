# ADR-017: Granular Approval Policy Enum for Shell Command Safety

## Status
Accepted

## Date
2026-05-12

## Context
Codex executes shell commands chosen by an AI model. The safety implications range from benign (reading files) to severe (deleting data, exfiltrating credentials). Users need fine-grained control over when they are prompted:
- **CI/automated environments**: never prompt; always execute
- **Interactive development**: prompt for potentially dangerous commands; auto-approve others
- **High-security environments**: prompt for everything
- **Trusted workflows**: auto-approve commands that match known patterns

A binary "prompt always / never prompt" switch is insufficient.

## Decision
Define `AskForApproval` enum in `codex-rs/protocol/src/` with variants:
- `UnlessTrusted`: auto-approve commands that match trusted patterns; prompt for others
- `OnFailure` (deprecated): prompt only after a command fails
- `OnRequest` (default): prompt unless the command has been pre-approved for the session
- `Granular`: per-command type approval (combines exec policy rules with interactive prompts)
- `Never`: execute all commands without prompting (dangerous; for CI use)

The `ReviewDecision` enum (returned by approval handlers) supports:
- `Approved` / `Denied` / `Abort`
- `ApprovedExecpolicyAmendment`: approve and update exec policy rules to remember the approval
- Session-scoped and prefix-scoped approvals

## Alternatives Considered

### Boolean allow/deny
- Pros: Simple
- Cons: Cannot express "always allow git commands but prompt for rm -rf"; cannot distinguish session-scoped from one-time approvals
- Rejected: Insufficient granularity for real workflows

### Capability-based permissions (POSIX-style)
- Pros: Fine-grained, composable
- Cons: Too complex for end-users to configure; requires understanding each capability
- Rejected: The enum variants map to recognizable user intent

## Consequences
- `--ask-for-approval <MODE>` CLI flag overrides config per-invocation
- `--dangerously-bypass-approvals-and-sandbox` disables all checks for CI use
- `AskForApproval::OnRequest` is the default — users are prompted unless they've approved a pattern for the session
- Exec policy `.rules` files (in `.codex/rules/`) provide persistent, reviewable approval patterns
- TUI shows approval keyboard shortcuts: `y` (once), `a` (session), `p` (prefix), `d` (deny)
