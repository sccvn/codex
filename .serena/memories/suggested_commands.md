# Suggested Commands

All `just` commands are run from the `codex-rs/` directory (justfile sets `working-directory := "codex-rs"`).

## Running
```sh
# Run the codex CLI
just codex [args]
# or: cargo run --bin codex -- [args]

# Run the TUI with exec-server
just tui-with-exec-server [args]

# Run MCP server
just mcp-server-run [args]

# Run app-server test client
just app-server-test-client [args]
```

## Building
```sh
# Build with Cargo
cargo build -p <crate-name>

# Build all (release, Bazel)
just build-for-release

# Run via Bazel
just bazel-codex [args]
```

## Formatting
```sh
# Format Rust code (run after any Rust changes)
just fmt
# This also formats Python SDK code with ruff
```

## Linting / Fixing
```sh
# Fix lint issues for a specific crate (preferred)
just fix -p <crate-name>

# Fix lint issues workspace-wide
just fix

# Just check (no fix)
just clippy
just clippy -p <crate-name>
```

## Testing
```sh
# Test a specific crate (preferred)
cargo test -p codex-<crate-name>

# Full test suite (requires cargo-nextest)
just test
# or: RUST_MIN_STACK=8388608 cargo nextest run --no-fail-fast

# Bazel tests
just bazel-test

# Install cargo-nextest if needed
cargo install --locked cargo-nextest

# Snapshot tests (insta)
cargo test -p codex-tui
cargo insta pending-snapshots -p codex-tui
cargo insta accept -p codex-tui
```

## Schema Generation
```sh
# Regenerate config.toml JSON schema
just write-config-schema

# Regenerate app-server protocol schema
just write-app-server-schema
# (add --experimental for experimental API fixtures)
```

## Bazel Lock
```sh
# Update Bazel lockfile after dependency changes
just bazel-lock-update

# Check Bazel lockfile for drift
just bazel-lock-check
```

## Install / Setup
```sh
just install
# Runs: rustup show active-toolchain && cargo fetch
```
