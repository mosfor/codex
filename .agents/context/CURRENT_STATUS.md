Project-scoped memory work for Codex CLI is partially implemented in this branch. The goal was to let each trusted project define its own SQLite state path and memory artifact path, while still keeping global `CODEX_HOME` for auth, shared skills, and similar global state.

What is done:
- Added `memory_home` to `Config` and `ConfigToml`.
- Default behavior stays the same: `memory_home` falls back to `$CODEX_HOME/memories`.
- Wired runtime memory read/write call sites to use `config.memory_home` instead of always deriving from `config.codex_home`.
- Updated memory consolidation sandbox writable roots so the subagent can write to `memory_home`, and also `codex_home` when those paths differ.
- Updated config tests for the new field.
- Regenerated `codex-rs/core/config.schema.json`.
- Documented the new config key in `docs/config.md`.

Important files:
- `codex-rs/core/src/config/mod.rs`
- `codex-rs/core/src/codex.rs`
- `codex-rs/core/src/memories/phase2.rs`
- `codex-rs/core/src/memories/prompts.rs`
- `codex-rs/core/src/memories/tests.rs`
- `codex-rs/core/src/state_db.rs`
- `codex-rs/core/src/config/config_tests.rs`
- `codex-rs/core/config.schema.json`
- `docs/config.md`

Important decisions:
- Keep `sqlite_home` separate from `memory_home`.
- Keep `CODEX_HOME` global by default.
- Let project config override `memory_home`, so project-local config like `.codex/config.toml` can use:
  - `sqlite_home = ".codex/state"`
  - `memory_home = ".codex/memory"`
- Remove the old noisy `state_db` mismatch warnings for the list APIs because `sqlite_home != codex_home` is now intentional.

Verification done:
- `just write-config-schema` completed successfully after switching to low-space build settings.
- `cargo fmt --all` completed successfully.
- `cargo test -p codex-core` was started, but not completed. It was interrupted because the build machine was running out of disk space.

Environment gotchas:
- This work was done on `/mnt/dvtest`, which is only about 9.8G total. That is too small for comfortable Rust workspace builds here.
- The first normal build failed with `No space left on device`.
- To keep going, the build used:
  - `RUSTUP_HOME=/mnt/dvtest/.rustup`
  - `CARGO_HOME=/mnt/dvtest/.cargo`
  - `CARGO_INCREMENTAL=0`
  - `CARGO_BUILD_JOBS=1`
  - `RUSTFLAGS='-C debuginfo=0'`
- Even with that, space dropped to a few hundred MB during testing.

What is still incomplete or unverified:
- Full `cargo test -p codex-core` result is unknown.
- `just fix -p codex-core` was not run here because the environment was already space-constrained.
- There may still be behavioral or test issues that only show up once the full test suite finishes on a machine with enough disk.

Where to restart first:
1. Move to a machine or partition with real headroom.
2. Re-run:
   - `just write-config-schema`
   - `cargo test -p codex-core`
3. If tests pass, run:
   - `just fix -p codex-core`
   - `just fmt`
4. Review whether any other `Config { ... }` test fixtures outside the touched area now need `memory_home`.
5. If needed, add one explicit integration-style test for project-local `.codex/config.toml` using both `sqlite_home` and `memory_home`.

Do not accidentally undo:
- The separation between `sqlite_home` and `memory_home`.
- The phase-2 writable-root fix that allows `memory_home` outside `codex_home`.
- The config schema and docs update for `memory_home`.

Best next actions by time budget:
- 15 minutes: run `cargo test -p codex-core` on a bigger machine and fix the first failure.
- 1 hour: get tests green, then run `just fix -p codex-core` and `just fmt`.
- Longer: add one more targeted test around trusted project config loading with repo-local memory/state paths, then open a PR.
