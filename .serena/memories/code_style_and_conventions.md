# Code Style and Conventions (Rust / codex-rs)

## Rust Edition & Formatting
- Rust edition: 2024
- Toolchain: 1.93.0 (see `codex-rs/rust-toolchain.toml`)
- rustfmt: `imports_granularity = "Item"` (each import on its own line)
- Run `just fmt` after any code changes

## Crate Naming
- All crate names are prefixed with `codex-` (e.g., `codex-core`, `codex-tui`)
- Avoid adding code to `codex-core` — it is already large; prefer new crates or existing smaller crates

## Code Style Rules (Clippy)
- Collapse nested `if` statements: `collapsible_if`
- Inline format args: `uninlined_format_args` — use `format!("{x}")` not `format!("{}", x)`
- Use method references over closures: `redundant_closure_for_method_calls`
- Avoid `bool` or ambiguous `Option` params; prefer enums, named methods, or newtypes
- Use `/*param_name*/` comments for opaque literal args (None, booleans, numbers) — `argument_comment_lint`
- Make `match` statements exhaustive — avoid wildcard arms
- Disallowed ratatui colors: `Color::Rgb`, `Color::Indexed`, `.white()`, `.black()`, `.yellow()`

## Async Traits
- Prefer native RPITIT: `fn foo(&self) -> impl std::future::Future<Output = T> + Send;`
- Avoid `#[async_trait]` and `#[allow(async_fn_in_trait)]`
- Implementations may use `async fn foo(&self) -> T`

## API / Protocol (app-server v2)
- Active API work in v2 only
- Payload naming: `*Params` (request), `*Response` (response), `*Notification` (notification)
- RPC methods: `<resource>/<method>`, singular resource (e.g., `thread/read`)
- camelCase fields on wire with `#[serde(rename_all = "camelCase")]`
- `#[ts(export_to = "v2/")]` on all v2 types
- No `#[serde(skip_serializing_if = "Option::is_none")]` for v2 payload fields (except specific exception)
- Optional fields in `*Params`: use `#[ts(optional = nullable)]`
- Timestamps: `i64` Unix seconds, named `*_at`
- Pagination: `cursor: Option<String>`, `limit: Option<u32>`, `data: Vec<...>`, `next_cursor: Option<String>`

## TUI / ratatui Style
- Use Stylize helpers: `.dim()`, `.bold()`, `.cyan()`, `.italic()`, `.underlined()`, `.red()`, `.green()`, `.magenta()`
- Prefer `"text".into()` for basic spans
- Avoid hardcoded `.white()` — use default foreground
- Chain helpers: `url.cyan().underlined()`
- Text wrapping: use `textwrap::wrap` for plain strings; use helpers in `tui/src/wrapping.rs` for ratatui `Line`s
- Styling: consult `codex-rs/tui/styles.md`

## Module Size
- Target modules under 500 LoC (excluding tests); hard limit ~800 LoC
- Prefer new modules over growing existing ones
- High-touch files to avoid extending: `tui/src/app.rs`, `tui/src/chatwidget.rs`, `tui/src/bottom_pane/`

## Tests
- Use `pretty_assertions::assert_eq` for clearer diffs
- Prefer deep equality comparisons over field-by-field
- TUI changes must include `insta` snapshot coverage
- Use `codex_utils_cargo_bin::cargo_bin("...")` to spawn workspace binaries in tests
- Avoid mutating process environment in tests

## Error Handling
- `large-error-threshold = 256` (Clippy config)
- `allow-expect-in-tests = true`, `allow-unwrap-in-tests = true`

## Docs
- New traits must have doc comments explaining role and usage
- No general product docs in `docs/` — official docs live elsewhere
- App-server API docs go in `app-server/README.md`
- Prefer private modules with explicit public API exports
