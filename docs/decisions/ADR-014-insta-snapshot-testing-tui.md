# ADR-014: insta Snapshot Testing for TUI Regression Detection

## Status
Accepted

## Date
2026-05-12

## Context
The TUI renders terminal frames as 2D character grids. Testing that the rendered output is correct is difficult with assertion-based unit tests because:
- The exact rendered output depends on widget composition, word wrapping, and styling
- Small changes to spacing or color can break rendering in non-obvious ways
- Visual regressions are hard to describe in code assertions

## Decision
Use `cargo-insta` for snapshot testing all TUI widget renders in `codex-rs/tui/tests/`. Snapshot tests capture the rendered terminal frame as a string and compare against a stored `.snap` file. Any UI change must include a snapshot update; updated snapshots are reviewed in PRs to make visual impact easy to audit.

Workflow:
1. `cargo test -p codex-tui` generates `.snap.new` files for changed snapshots
2. `cargo insta pending-snapshots -p codex-tui` lists pending changes
3. `cargo insta accept -p codex-tui` accepts all new snapshots after review

## Alternatives Considered

### Pure assertion tests (assert_eq! on rendered strings)
- Pros: No external tooling
- Cons: Writing the expected string inline is tedious and brittle; changes require updating many string literals; no visual diff
- Rejected: Snapshot files are more maintainable and provide better diffs

### Screenshot testing (image comparison)
- Pros: True visual comparison
- Cons: Terminal screenshots vary across environments (font rendering, terminal emulator); high flakiness; large binary artifacts in git
- Rejected: Text-based snapshots are deterministic and portable

### No TUI tests
- Pros: Less maintenance
- Cons: Regressions in transcript rendering, word wrapping, and modal overlays go undetected
- Rejected: Visual regressions are a real, observed problem

## Consequences
- `cargo-insta` is a required dev dependency (`cargo install --locked cargo-insta`)
- Every PR that changes visible TUI output must include accepted snapshot updates
- Snapshot files (`.snap`) are committed to `codex-rs/tui/src/snapshots/`
- `pretty_assertions::assert_eq` is used in all test modules for clearer diffs
