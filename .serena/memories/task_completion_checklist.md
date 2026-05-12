# Task Completion Checklist

After making Rust code changes in `codex-rs`:

1. **Format** (always, no approval needed):
   ```sh
   just fmt
   ```

2. **Fix lint** (scoped preferred):
   ```sh
   just fix -p codex-<crate-name>
   # Only run workspace-wide if shared crates (core, protocol) were changed:
   just fix
   ```

3. **Test** (run tests for affected crate first):
   ```sh
   cargo test -p codex-<crate-name>
   # If common/core/protocol changed, ask user before running full suite:
   just test
   ```

4. **Snapshot tests** (if TUI changed):
   ```sh
   cargo test -p codex-tui
   cargo insta accept -p codex-tui   # only after reviewing .snap.new files
   ```

5. **Schema updates** (if config types changed):
   ```sh
   just write-config-schema
   ```

6. **Schema updates** (if app-server protocol types changed):
   ```sh
   just write-app-server-schema
   # Add --experimental if experimental API fixtures affected
   cargo test -p codex-app-server-protocol
   ```

7. **Bazel lockfile** (if Cargo.toml / Cargo.lock changed):
   ```sh
   just bazel-lock-update
   just bazel-lock-check
   ```

8. **Do NOT run `just fix` or `just fmt` after tests** — only before finalizing.

## Notes
- Do not add `--all-features` to routine test runs (expands build matrix)
- Be patient with Rust commands — Rust lock can make execution slow
- Do not kill running Rust commands by PID
- Do not add code to `codex-core` unless necessary
- When adding `include_str!` or similar, update `BUILD.bazel` compile_data
