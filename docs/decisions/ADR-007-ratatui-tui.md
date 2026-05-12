# ADR-007: ratatui for Terminal User Interface

## Status
Accepted

## Date
2026-05-12

## Context
Codex needs a rich interactive terminal UI with:
- Multi-pane layout (transcript above, composer below, modals overlaid)
- Incremental rendering (only repaint changed regions to avoid flicker at 60fps)
- Cross-platform terminal compatibility (xterm, iTerm2, Windows Terminal, tmux, Zellij)
- Composable widgets (chat composer, approval modal, file search popup, slash command popup)
- Snapshot testing for UI regression detection

## Decision
Use `ratatui` (a maintained fork of `tui-rs`) as the terminal UI framework. UI is structured as a tree of `Widget` implementations. The main event loop in `codex-rs/tui/src/app.rs` drives frame rendering on each terminal event or agent event. Styling uses ratatui's `Stylize` trait helpers exclusively (`.red()`, `.bold()`, `.dim()`) rather than constructing `Style` objects manually.

## Alternatives Considered

### cursive
- Pros: Higher-level abstraction, event-driven model
- Cons: Less control over layout and rendering; widget library is smaller; limited support for arbitrary Unicode/emoji rendering
- Rejected: Codex's transcript rendering (code blocks, diffs, reasoning spans) requires fine-grained control

### crossterm + raw terminal
- Pros: Maximum control
- Cons: Building layout, word wrapping, and widget composition from scratch is a large investment; no snapshot testing primitives
- Rejected: ratatui provides all needed primitives without sacrificing control

### Electron / web-based UI
- Pros: Rich styling, existing web component ecosystem
- Cons: Large binary size; requires Electron runtime; loses the "terminal-native" UX; incompatible with SSH/remote usage
- Rejected: Terminal-native UX is a core product identity

## Consequences
- All TUI styling must use Stylize helpers; hardcoded `.white()` is prohibited (breaks light terminal themes)
- Text wrapping uses `textwrap::wrap` for plain strings; `word_wrap_lines()` for ratatui `Line`s (from `codex-rs/tui/src/wrapping.rs`)
- UI changes must include `insta` snapshot test updates (`cargo insta accept -p codex-tui`)
- `--no-alt-screen` flag disables the alternate screen buffer for compatibility with Zellij and other terminal multiplexers
