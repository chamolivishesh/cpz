# ARCHITECTURE

## 1. Pipeline

Planning, conflict resolution, and execution are separate stages connected by bounded channels.

```text
CLI/config ─► Planner ─► Conflict stage ─► Executor (workers) ─► Reporter
               (scan)     (serial, owns       (copy/move/verify)   (progress,
                           the terminal)                             log, json)
```

1. **Planner** validates arguments (identity, destination-inside-source, overlapping sources, colliding targets incl. case-folding) and walks the trees, emitting `Op` items (`CopyFile`, `MakeDir`, `Symlink`, `HardLink`, `Skip`, `Remove`, ...). It also computes totals for the progress bar; the totals arrive asynchronously, so the reporter must handle "unknown total yet".
2. **Conflict stage** receives ops that hit an existing destination entry. It is the **only** component that ever prompts, which removes terminal-lock problems: workers never touch stdin. Non-conflicting ops pass straight through. Its decisions are deterministic and testable by injecting a scripted `Prompter`.
3. **Executor** is a bounded worker pool. Each worker performs one op at a time using the safe-write sequence in SPEC §9, checks the cancel flag between chunks, and sends progress events.
4. **Reporter** consumes events, renders progress to stderr, writes logs and JSON, and computes the final summary and exit code. Progress is suspended while the conflict stage has a prompt open.

Phase 1 runs the executor with one worker. The structure does not change when `--jobs` arrives.

## 2. Bounded memory

- Channels are bounded; the planner blocks when workers fall behind.
- The planner does a cheap first pass for totals (stat only) and streams ops; it does not hold the whole tree.
- Interning or compact path storage if profiling shows path memory dominating.
- Dry-run and `sync --delete` need the full plan before acting. Spill to a temporary on-disk file above a size threshold (reuse the journal format).
- Buffers: pooled, `jobs × buffer_size`, allocated once.

## 3. Crate and module layout

```text
Cargo.toml                 # workspace; [workspace.lints]
crates/
  cpz/                     # binary + library (unsafe_code = "deny")
    src/
      main.rs              # init, signals, exit code only
      lib.rs
      cli.rs  config.rs  errors.rs
      plan/    (mod.rs, scan.rs, safety.rs)
      conflict/ (mod.rs, policy.rs, prompt.rs, naming.rs)
      exec/    (mod.rs, copy.rs, mover.rs, verify.rs, metadata.rs, symlinks.rs)
      report/  (mod.rs, progress.rs, log.rs, json.rs)
      filter.rs  journal.rs  sync.rs
      platform/ (mod.rs, unix.rs, linux.rs, macos.rs, windows.rs)
  cpz-sys/                 # only if raw unsafe is ever unavoidable; otherwise absent
```

Split into directories as modules grow; do not create empty modules for future phases.

## 4. Key abstractions

- `trait Fs`: the small set of filesystem operations the executor needs (open, rename, metadata, create_temp, ...). The real impl uses std/rustix; the test impl injects `EXDEV`, write errors, `ENOSPC`, and read failures. This is how cross-device `move` and cleanup-on-failure are tested without root or special mounts.
- `trait Prompter`: `fn ask(&mut self, conflict: &Conflict) -> Decision`. Real impl reads the TTY; test impl is scripted.
- `enum Policy` and `struct ConflictState`: policy plus remembered "all" choices.
- `struct Op` / `enum OpResult`: error variants carry operation, paths, and source `io::Error`.
- `struct CancelToken(Arc<AtomicBool>)`.
- Progress counters are `AtomicU64`; the reporter samples them on a timer (about 10 Hz). Do not update a lock per syscall.

## 5. Copy engine

Order of preference per file:
1. Reflink clone (`--reflink=auto`): `FICLONE` on Linux, `clonefile` on macOS.
2. `copy_file_range` (Linux) / `CopyFileEx` (Windows) where no inline hashing, chunk-level progress, or cancellation granularity is needed. **Caveat:** kernel fast paths give coarse progress, so for the progress UI the default is to use `copy_file_range` in bounded chunks (for example 4–64 MiB per call) and check cancellation and update counters between calls. Benchmark this against the plain read/write loop before choosing the default.
3. User-space loop with pooled buffer: needed for inline blake3 (`--verify`), sparse handling when the kernel path cannot, and the fallback when others fail with `EXDEV`/`EINVAL`/`ENOSYS`.

Handle `Interrupted`, partial writes, and zero-byte files. Detect source growth/truncation. Check source/destination identity before opening the temp file.

## 6. Syscalls without `unsafe`

The main crate denies `unsafe`. Use safe wrappers:
- `rustix` for `copy_file_range`, `renameat2`-style no-replace rename, `seek` data/hole, `fcntl`, `ioctl` clone, as far as it exposes them safely on each platform. Verify per-platform availability when implementing.
- `filetime` for timestamps; `same-file` (or `rustix` stat) for identity.
- If something is missing, ask before adding `unsafe` or a `cpz-sys` crate.

Note on `#![forbid(unsafe_code)]`: `forbid` cannot be locally overridden, which is why this project uses `deny` plus a possible separate crate instead.

## 7. Concurrency rules

- A destination path (and its temp path) is owned by at most one worker. Collisions are resolved in planning, not by locking at write time.
- Directory metadata (times, permissions) is applied **after** all children are written (post-order), by a single finalizer, because creating children changes the directory mtime. Read-only directories are created writable and fixed up at the end.
- Use `std::thread` with bounded `crossbeam-channel`. `rayon` is a poor fit for blocking I/O workers.

## 8. Dependencies (target set)

Chosen to be small and Debian-packageable. Check each against the Debian archive before use.

| Need | Crate |
|---|---|
| CLI | `clap` (derive) |
| errors | `thiserror`; `anyhow` in `main.rs` only |
| progress | `indicatif` (+ `console`) |
| walking | `walkdir` |
| globbing | `globset` |
| hashing | `blake3` |
| syscalls | `rustix` |
| times / identity | `filetime`, `same-file` |
| temp files | `tempfile` |
| logging | `tracing`, `tracing-subscriber` |
| JSON / journal | `serde`, `serde_json` |
| channels | `crossbeam-channel` |
| signals | `signal-hook` (or `ctrlc`) |
| tests | `assert_cmd`, `predicates`, `proptest` |

Do not add `rayon`, `ignore`, or `xxhash-rust` unless a measured need appears.

## 9. Testing strategy

See also AGENTS.md. In short: unit tests per module; integration tests via `assert_cmd` with `TempDir` fixtures; fake `Fs`/`Prompter` for failure injection; `proptest` for naming, path-safety, and filter logic; fuzz the journal parser (`cargo fuzz`) once it exists; CI on Linux, macOS, Windows. TTY behavior (prompts, progress) is tested with a pseudo-terminal crate (such as `expectrl`) in a small number of tests.
