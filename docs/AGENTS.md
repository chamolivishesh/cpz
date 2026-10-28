# AGENTS.md

Instructions for AI coding agents working in this repository. Keep this file short; details live in `docs/`.

## What this project is

`cpz` (working name, see `docs/PACKAGING.md`) is a Rust file-copy tool for Linux (macOS and Windows secondary) that behaves like `cp` for basic use but adds:

- live progress: current file (name, path, per-file bar) plus overall job progress, speed, ETA
- interactive conflict resolution (overwrite / skip / rename / keep both / compare contents)
- `move`, verification, dry runs, and many optional flags

Target users: people who find `cp` silent. Target distribution: GitHub releases, then the Debian archive.

## Read first

| Doc | Read when |
|---|---|
| `docs/SPEC.md` | changing any user-visible behavior, flag, exit code, or prompt |
| `docs/ARCHITECTURE.md` | adding modules, touching the pipeline, threading, or syscalls |
| `docs/ROADMAP.md` | deciding what to build next; **current phase is stated at the top** |
| `docs/PACKAGING.md` | adding a dependency, changing the license, naming, man page, or CI |

If code and `docs/SPEC.md` disagree, stop and ask. Do not silently pick one.

## Hard invariants (never violate)

1. **No unexpected data loss.** Never overwrite, delete, or truncate user data unless a documented policy or an explicit user answer says to. The destination is only ever replaced by an atomic rename of a fully written temp file.
2. **Never report success early.** Exit 0 and "done" messages only after every required operation (and verification, if requested) finished.
3. **Never delete a source** (`move`) until the destination is fully written, flushed, and verified.
4. **Paths are `Path`/`OsStr`, never `String`.** No `.to_str().unwrap()`. Non-UTF-8 names must work.
5. **Plan before mutating.** Validate source/destination identity, destination-inside-source, and colliding targets before the first write.
6. **Nothing hangs on special files.** Never `read` a FIFO or socket.
7. **Errors carry context:** operation + path(s) + OS error. No swallowed I/O errors.
8. **No interactive prompt without a TTY.** Without one, require an explicit policy or exit with code 4.

## Commands

Run before declaring any task done:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
cargo test --doc
cargo build --release
```

Also, when touching platform code: `cargo check --target x86_64-pc-windows-gnu` (and `x86_64-apple-darwin` if the target is installed). You cannot run those tests locally; say so in your summary instead of claiming they pass.

Run one test: `cargo test <name> -- --nocapture`. Heavy tests are `#[ignore]`d: `cargo test -- --ignored`.

## Conventions

- **Unsafe:** the main crate has `unsafe_code = "deny"`. Use safe wrappers (`rustix`, `filetime`, `same-file`) instead of raw syscalls. If a raw syscall is truly unavoidable, it goes in the separate `cpz-sys` crate with a safety comment and its own tests. Ask first.
- **Lints** are configured in `[workspace.lints]`: `unwrap_used` and `expect_used` are `deny` in non-test code. Do not add `#[allow]` to get around them; fix the code.
- **Errors:** `thiserror` in library code, `anyhow` only in `main.rs`.
- **Dependencies:** every new crate must (a) be justified in the PR, (b) have a permissive license, and (c) be packageable by Debian. Prefer std. Ask before adding one. See `docs/PACKAGING.md`.
- **Platform code** lives behind `cfg` in `src/platform/`. No `cfg` scattered through core logic.
- **Output:** progress and prompts go to **stderr**; machine output (`--json`) goes to **stdout**. No ANSI codes when not a TTY.
- **Tests:** every behavior change needs a test. Use `tempfile::TempDir`; never touch `$HOME` or fixed paths. No multi-GB files in default test runs.
- **Scope:** small, reviewable changes within the current roadmap phase. Do not rewrite unrelated code. No `todo!()` or stubs that return `Ok(())`.

## Ask before

- adding a dependency or any `unsafe`
- changing a flag name, default, exit code, or prompt wording
- deleting or renaming public modules
- implementing anything outside the current roadmap phase
- anything that deletes files outside `sync --delete` or `move`

## Final summary format

List: files changed, exact commands run and their results, platforms **not** tested, known edge cases or follow-ups.
