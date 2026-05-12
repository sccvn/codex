# ADR-015: Policy to Resist Growing codex-core

## Status
Accepted

## Date
2026-05-12

## Context
`codex-rs/core/` is the largest crate in the workspace. Because it is the most "central" crate, new features are frequently added there rather than to smaller, more focused crates. This creates a negative feedback loop:
- `codex-core` becomes slower to compile
- Its dependency tree grows, making it harder to use individual pieces in isolation
- Unrelated concepts are co-located, violating single-responsibility

Examples of code that was added to `codex-core` but should have been in separate crates: sandboxing utilities, MCP connection management, exec policy logic.

## Decision
Establish an explicit policy: **resist adding code to `codex-core`**. Before adding any new code to `codex-core`, ask:
1. Is there an existing crate other than `codex-core` that is an appropriate home?
2. Should a new crate be introduced for this functionality?

Additionally, target all module files at under 500 LoC and require new modules instead of extending files approaching 800 LoC. High-touch files are specifically called out in `CLAUDE.md`: `app.rs`, `chatwidget.rs`, `chat_composer.rs`, `footer.rs`, `mod.rs` (bottom_pane).

## Alternatives Considered

### No policy, organic growth
- Pros: No friction for contributors
- Cons: `codex-core` compile times continue to grow; dependency isolation degrades
- Rejected: Organic growth has already caused the problem; a policy is needed

### Enforce with automated tooling (depcheck, module size linter)
- Pros: Automated enforcement
- Cons: Tooling not yet available; policy can be documented and enforced in code review now
- Noted: Automated enforcement is a future improvement

## Consequences
- Code review must flag PRs that add substantial new code to `codex-core` without justification
- New features (new concepts, new subsystems) should start in new crates
- Existing overloaded modules in `codex-core` should be refactored to sub-modules when touched
- This policy is documented in `CLAUDE.md` and `.serena/memories/code_style_and_conventions.md`
