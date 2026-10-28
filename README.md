# cpz

> **Working name.** The final name is not decided yet; see [docs/PACKAGING.md](docs/PACKAGING.md).

A fast, safe file-copy tool for Linux that tells you what it is doing.

`cp` copies silently. `cpz` shows the current file, its progress, and the progress of the whole job, and asks you what to do when a file already exists. For basic use it is as simple as `cp`; everything else sits behind optional flags.

> **Status: pre-alpha. Nothing described below is released yet.** This README describes the target behavior defined in [docs/SPEC.md](docs/SPEC.md). Features are marked by roadmap phase in [docs/ROADMAP.md](docs/ROADMAP.md). Do not use this on irreplaceable data until a tagged release says it is ready.

## What it will look like

```text
Copying 1,204 files (38.2 GiB)               elapsed 02:14
  ▸ vacation/IMG_2041.CR2
    from /home/me/photos  →  /mnt/usb/photos
    [████████████████████████░░░░░░░░░░░░░░░░] 61%   18.4/30.1 MiB   212 MiB/s
    
  Overall [████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░] 31%  11.9/38.2 GiB  avg 188 MiB/s  ETA 02:31
  files 372/1204   skipped 4   failed 0
```

When a destination file already exists:

```text
Conflict: report.pdf
  Source:      8.4 MiB   modified 2026-10-27 18:02   /home/me/new/report.pdf
  Destination: 7.9 MiB   modified 2026-10-20 09:41   /backup/report.pdf

  [o] overwrite        [s] skip            [r] rename incoming (type a new name)
  [k] keep both        [d] compare contents  [u] overwrite only if newer
  [O] overwrite all    [S] skip all        [K] keep both for all
  [q] quit
>
```

## Usage

```bash
cpz big.iso /mnt/usb/                 # copy a file
cpz photos/ /backup/                  # copy a directory (recursive by default)
cpz -T photos /backup/photos          # merge contents into an existing directory
cpz a.txt b.txt docs/ /backup/        # several sources
cpz move old/ /archive/               # move (safe copy-verify-delete across disks)
cpz --verify -a project/ /backup/     # archive mode, check contents after copying
cpz --dry-run photos/ /backup/        # show the plan, change nothing
cpz -n photos/ /backup/               # skip files that already exist
cpz -f photos/ /backup/               # overwrite files that already exist
```

Path rules follow GNU `cp`: `cpz dir dest/` creates `dest/dir/`. Trailing slashes do not change meaning; use `-T` to merge contents.

### Common flags

| Flag | Meaning |
|---|---|
| `-a`, `--archive` | recursive; preserve links, permissions, timestamps |
| `-p`, `--preserve[=LIST]` | preserve `mode`, `times`, `links` |
| `-i`, `--interactive` | always prompt on conflicts |
| `-f`, `--overwrite` | replace existing files |
| `-n`, `--skip-existing` | leave existing files alone |
| `-u`, `--update` | replace only if the source is newer |
| `--keep-both` | rename the incoming file (`report (1).pdf`) |
| `--on-conflict=POLICY` | `ask`, `overwrite`, `skip`, `rename`, `keep-both`, `fail` |
| `--verify` | re-read and compare after copying |
| `--include PAT`, `--exclude PAT` | glob filters |
| `--dry-run` | plan only |
| `-j`, `--jobs N` | parallel copies |
| `--no-progress`, `-q`, `-v` | output control |
| `--json` | machine-readable events on stdout |

The complete list, defaults, and rejected flag combinations are in [docs/SPEC.md](docs/SPEC.md).

### Scripts and non-interactive use

Prompts appear only when stdin and stderr are terminals. Otherwise you must pass a conflict policy (for example `-n` or `-f`), or the run stops with exit code 4. Progress goes to stderr; `--json` output goes to stdout.

| Exit code | Meaning |
|---|---|
| 0 | success |
| 1 | fatal error |
| 2 | usage error |
| 3 | partial failure |
| 4 | unresolved conflicts |
| 5 | verification mismatch |
| 130 / 143 | interrupted (SIGINT / SIGTERM) |

## Safety design

- Files are written to a temporary file in the destination directory and moved into place with an atomic rename. A half-copied file never replaces a good one.
- Nothing is overwritten or deleted without a policy or an explicit answer from you.
- `move` deletes the source only after the destination is written and verified.
- Paths are handled as raw OS paths, so non-UTF-8 filenames work.
- The source and destination are checked for identity, nesting, and name collisions before anything is written.

Limits worth knowing: `--verify` reads the file back, which usually comes from the page cache, so it catches copy bugs and many I/O errors but is not proof against media corruption. Atomic rename is not crash durability; use `--fsync` for that. Details are in [docs/SPEC.md](docs/SPEC.md) §9.

## Compared with other tools

Other tools cover parts of this, and they may suit you better today:

- **`cp`** (GNU coreutils): ubiquitous and exact, but silent.
- **`rsync`**: powerful, including delta and network transfer; progress output with `--info=progress2` is available but shows only overall progress, and the command line is long.
- **Patched coreutils** (such as `advcpmv`) and other Rust copy tools (such as `xcp`): worth checking before you choose.

The aim here is the combination: no-setup per-file and overall progress, built-in interactive conflict handling, and `cp`-style flags. This is not a replacement for `rsync` over a network, and `sync` is a later phase.

## Install

Not available yet. Planned, in order: build from source, prebuilt binaries on GitHub Releases, `cargo install`, then packages for Linux distributions including Debian ([docs/PACKAGING.md](docs/PACKAGING.md)).

Build from source (once Phase 0 lands):

```bash
git clone https://github.com/<you>/<repo>.git
cd <repo>
cargo build --release
./target/release/cpz --help
```

## Documentation

| File | Contents |
|---|---|
| [docs/SPEC.md](docs/SPEC.md) | behavior, flags, prompts, exit codes |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | pipeline, modules, dependencies, testing |
| [docs/ROADMAP.md](docs/ROADMAP.md) | phases and exit criteria |
| [docs/PACKAGING.md](docs/PACKAGING.md) | naming, licensing, release, Debian |
| [AGENTS.md](AGENTS.md) | rules for AI coding agents working in this repo |

## Contributing

Issues and pull requests are welcome once the Phase 0 skeleton exists. Before opening a PR, run:

```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
```

Changes to flags, defaults, or exit codes must update [docs/SPEC.md](docs/SPEC.md) in the same PR.

## License

To be decided before the first public commit; see [docs/PACKAGING.md](docs/PACKAGING.md). Placeholder: MIT OR Apache-2.0.
