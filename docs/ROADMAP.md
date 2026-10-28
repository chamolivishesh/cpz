# ROADMAP

**Current phase: 0 (not started).** Update this line when a phase completes.

Phases are ordered by real dependencies. Do not start a phase until the previous one's exit criteria pass. The first public release is **v0.1 at the end of Phase 2**: it is already a useful `cp` with progress and conflict prompts. Everything after that is additive.

## Phase 0: Skeleton
- Cargo workspace, `[workspace.lints]` (`unsafe_code = deny`, `unwrap_used`/`expect_used = deny` outside tests), pinned edition and MSRV, `rustfmt.toml`.
- CI on Linux (fmt, clippy, test, doc tests, release build) plus `cargo check` for Windows and macOS targets.
- `LICENSE`, `README` stub, `cpz --version`, `--help`.
- Decide the final binary name (see PACKAGING).

**Exit:** CI green on an empty-but-real binary.

## Phase 1: Safe single-threaded copy, with progress
- CLI for `cpz SRC... DEST`, `-T`, `-t`, path semantics from SPEC §3.
- Planner with path-safety checks (identity, destination-inside-source, colliding targets, case-fold collisions).
- Executor: temp file → copy → rename; zero-byte files; size-change detection; cleanup on error.
- Recursive copy, directory merge, symlinks (copy link, `-L`/`-H`), special-file skipping, post-order directory metadata.
- Progress UI per SPEC §6 (per-file + overall, speed, ETA, non-TTY fallback).
- Ctrl+C handling with temp cleanup, exit codes 0/1/2/3/130/143.
- `trait Fs` seam for failure injection.

**Exit:** copies a mixed tree correctly; a killed copy leaves no partial *destination* files (temp files only, cleaned on graceful cancel); progress verified manually on a real terminal with a multi-GB file.

## Phase 2: Conflicts, verify, filters → **v0.1 release**
- Conflict stage with `Prompter`, all policies, keep-both naming, "apply to all", compare-contents (SPEC §5).
- Exit code 4 for unresolved non-interactive conflicts.
- `--verify` (inline blake3 on the user-space path) and `cpz verify`; exit code 5.
- `--include`/`--exclude`, `--dry-run`, `-p`/`-a` metadata preservation, `--hard-links`.
- `-v`, `--log`, `--json`.
- `move`, including EXDEV copy-verify-delete (verification now exists, so this is safe).
- Man page, shell completions (`clap_complete`), README with honest comparison.

**Exit:** all SPEC §5 behaviors covered by tests including scripted prompts and a pseudo-terminal test; stale temp-file detection; man page builds.

## Phase 3: Performance
- Benchmarks first (many small files, few huge files; against `cp` and `rsync`).
- Kernel fast paths in chunks (`copy_file_range`), reflink, sparse handling.
- `--jobs N` worker pool, buffer pool, bounded backpressure, multi-row progress.
- Change defaults only if benchmarks justify it.

**Exit:** benchmark results recorded in `docs/BENCHMARKS.md`; no regression in Phase 1–2 tests.

## Phase 4: Sync
- Diffing, `--update`, `--checksum`, explicit `--delete` with filter protection, bottom-up deletion, plan spill to disk for large trees.
- Sync safety tests (SPEC §14).

## Phase 5: Resume and extras (optional)
- Journal and `cpz resume`, with corruption and tamper tests (and fuzzing).
- `inspect` (only if still wanted).
- Ownership/xattr/ACL preservation behind explicit flags.

## Parking lot
Ideas that are not scheduled: bandwidth limit, desktop notification on finish, TUI mode, config file.
