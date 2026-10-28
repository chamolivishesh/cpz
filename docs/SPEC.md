# SPEC: User-visible behavior

This is the source of truth for CLI behavior. Binary name is `cpz` (working name).

## 1. Goals and non-goals

**Goals**
- Basic use is as simple as `cp`: `cpz a.iso /mnt/usb/`.
- You always see what is happening: current file, per-file progress, overall progress, speed, ETA.
- Conflicts are handled interactively by default, with clear choices.
- Optional power features sit behind flags and never change default behavior.
- Safe by default with irreplaceable data.

**Non-goals (for now)**
- Network transfer or delta sync (use `rsync`).
- Replacing `cp` in scripts that rely on exact GNU semantics.
- ACLs, xattrs, ownership, Windows ADS (post-1.0, behind explicit flags).

**Positioning.** Alternatives exist, and the README should say so honestly: `rsync -ah --info=progress2`, patched coreutils (`advcpmv`), and other Rust copy tools such as `xcp`. The pitch for this tool is the combination: zero-config progress with per-file and overall bars, built-in interactive conflict handling, and `cp`-familiar flags. Check what the others currently do before writing comparison claims.

## 2. Commands

```bash
cpz [copy] <SOURCE...> <DEST>     # `copy` is the default subcommand
cpz move   <SOURCE...> <DEST>
cpz verify <SOURCE> <DEST>
cpz sync   <SOURCE> <DEST>        # later phase
cpz inspect <PATH...>             # later phase, optional
cpz resume <JOB_ID>               # later phase
```

`cpz a b` must work with no subcommand.

## 3. Path semantics (follows GNU `cp`)

| Command | Result |
|---|---|
| `cpz file dest/` (dest is a dir) | `dest/file` |
| `cpz file newname` | file copied to `newname` |
| `cpz dir dest/` (dest exists) | `dest/dir/` created, contents merged if it already exists |
| `cpz dir newdir` (newdir missing) | `newdir/` created as a copy of `dir` |
| `cpz -T dir dest` | contents of `dir` merged directly into `dest` (no `dest/dir`) |
| `cpz -t dest a b c` | explicit target directory form |
| `cpz a b c dest` | `dest` must be an existing directory, else exit 2 |

- Trailing slashes do **not** change meaning (unlike rsync). Use `-T` to merge contents.
- **Directories are recursive by default** (a deliberate usability deviation from `cp`). `-r`/`-R` are accepted as no-ops for muscle memory.
- Merging preserves unrelated destination entries. Never remove anything during `copy`.

## 4. Defaults

| Topic | Default | How to change |
|---|---|---|
| Symlinks in a tree | copy the link itself (like `cp -r`) | `-L` follow all, `-H` follow command-line only |
| Dangling symlinks | copied as links, no error | n/a |
| Symlink loops | detected when following; skipped with warning | n/a |
| Permissions | like `cp`: source mode masked by umask | `-p` / `--preserve=mode` |
| Timestamps | not preserved (like `cp`) | `-p` / `--preserve=times` |
| `-a`, `--archive` | recursive + preserve links, mode, times | |
| Hard links | copied as independent files | `--hard-links` |
| Special files (FIFO, socket, devices) | skipped with a warning, counted in summary | n/a |
| Sparse files | `--sparse=auto` | `never` / `always` |
| Reflink | `--reflink=auto` | `never` / `always` |
| Verification | off | `--verify` |
| fsync | off | `--fsync` |
| Progress | on when stderr is a TTY | `--no-progress` |
| Conflicts | prompt if stdin and stderr are TTYs, else error | see §5 |

## 5. Conflict resolution

A conflict is an existing destination entry at a path the job wants to write.

Detect with `symlink_metadata` (never follow unintentionally). Handle files, directories, and symlinks as distinct types.

### Policies (non-interactive)
`--overwrite` (`-f`), `--skip-existing` (`-n`), `--keep-both`, `--update` (`-u`, replace only if source is strictly newer; compare mtimes with 2-second tolerance on FAT/exFAT, otherwise nanosecond), `--no-clobber` (alias of `--skip-existing`; see note), `--on-conflict=<ask|overwrite|skip|rename|keep-both|fail>`.

- `--no-clobber` is an alias of `--skip-existing`. For fail-on-conflict use `--on-conflict=fail`.
- Passing two different policies is a usage error (exit 2). `--on-conflict` and the shortcut flags are mutually exclusive.
- `-i`/`--interactive` forces prompting (and errors if no TTY).

### Interactive prompt

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

- `[d]` hashes both files (blake3) or compares byte by byte, shows identical/different, then re-prompts with an additional `[i] identical, skip` option. Never hash by default.
- Equal size is never proof of equal contents.
- "All" choices apply only to this job and only to conflicts of the same kind (file vs file).
- Keep-both naming: `report (1).pdf`, `report (2).pdf`, ... verifying the candidate does not exist. Extension handling: `archive.tar.gz` becomes `archive (1).tar.gz`.
- `[r]` rejects empty names, names containing `/`, `.`/`..`, and names that themselves collide (re-prompt).
- Stdin EOF or Ctrl+D at a prompt acts as `[q]`.
- Prompts are **serialized**: only the conflict-resolution stage prompts (see ARCHITECTURE). Progress rendering is suspended while a prompt is displayed.

### Hard rules
- Never replace a non-empty directory with a file, or a directory with a file.
- Never write to a destination that is the same file as the source (device+inode).
- Destination is a symlink: the **link itself** is the conflict. Overwrite replaces the link, not its target. Say so in the prompt.
- Destination is read-only: treated as a conflict like any other. Overwrite works via replace-by-rename where the directory permits it, otherwise error.

## 6. Progress display

Rendered to **stderr**, only when stderr is a TTY and `--no-progress` / `--json` / `--quiet` are absent.

```text
Copying 1,204 files (38.2 GiB)               elapsed 02:14
  ▸ vacation/IMG_2041.CR2
    from /home/me/photos  →  /mnt/usb/photos
    [████████████████████████░░░░░░░░░░░░░░░░] 61%   18.4/30.1 MiB   212 MiB/s

  Overall [████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 31%  11.9/38.2 GiB  avg 188 MiB/s  ETA 02:31
  files 372/1204   skipped 4   failed 0
```

Requirements:
- Show current file name and source/destination directories; with `--jobs N` show up to N active rows.
- Per-file bar, overall bar, bytes done / total, current and average throughput, ETA, elapsed, file counts, failures.
- Truncate long paths in the middle with `…`; never wrap or corrupt on narrow terminals; reflow on resize.
- Zero-byte files and empty jobs must not divide by zero.
- While the total is still being scanned, show a spinner and "scanning…", not a fake percentage.
- Report copied bytes and verified bytes separately when `--verify` is on.
- Final summary on stderr: files copied, skipped, failed, bytes, time, average speed. Exit code reflects failures.
- Non-TTY fallback: no bars. With `-v`, one line per file. Otherwise only the summary.

## 7. Flags

Short flags mirror `cp` where one exists, so users do not relearn them.

| Flag | Meaning |
|---|---|
| `-r`, `-R` | accepted; recursion is already default |
| `-a`, `--archive` | recursive, preserve links/mode/times |
| `-p`, `--preserve[=LIST]` | `mode,times,links` (ownership, xattr, acl later) |
| `-L`, `-H`, `-P` | symlink following policy (see §4) |
| `-T`, `--no-target-directory` | merge contents into DEST |
| `-t`, `--target-directory DIR` | explicit target |
| `-i`, `--interactive` | force prompts |
| `-f`, `--overwrite` | replace conflicts |
| `-n`, `--skip-existing` | skip conflicts |
| `-u`, `--update` | replace only if newer |
| `--keep-both` | rename incoming on conflict |
| `--on-conflict=POLICY` | explicit policy |
| `--checksum` | decide "same" by content hash instead of size+mtime (where the comparison is relevant) |
| `--verify` | re-read and compare after copying |
| `--fsync` | durability: fsync file and parent directory |
| `--hard-links` | preserve hard-link relationships within the job |
| `--reflink=auto\|always\|never` | CoW clone |
| `--sparse=auto\|always\|never` | hole handling |
| `--include PAT`, `--exclude PAT` | glob filters; later flags take precedence, first match wins |
| `--dry-run` | print the plan, change nothing |
| `-j`, `--jobs N` | parallel file copies (default 1 until Phase 4 benchmarks decide) |
| `--buffer-size SIZE` | user-space buffer (default 4 MiB, bounds 64 KiB to 64 MiB) |
| `--no-progress`, `-q`, `-v` | output control |
| `--json` | JSON-lines events on stdout |
| `--log FILE` | write a log file |
| `--resume`, `cpz resume ID` | resume a journaled job (later phase) |

Logging: `-v`, `-vv` control stderr verbosity; `--log FILE` writes the same events to a file; `--json` changes stdout format. There is no separate `--detailed-logs`.

Rejected combinations (exit 2): conflicting policies; `--interactive` with `--json`; `--json` combined with prompts (JSON mode requires a non-interactive policy); `--sparse=always` with `--reflink=always` when the filesystem cannot do both; `--hard-links` with `-L`.

## 8. Move semantics

- Same filesystem: `rename`, honoring conflict policy (use no-replace rename where the platform supports it to avoid races).
- Different filesystem (`EXDEV`): copy to temp, apply metadata, verify (size + content hash), atomic rename into place, then delete the source. Directories: delete source files as each is verified, remove source directories bottom-up only when empty.
- If any file fails, keep its source and report partial failure (exit 3). Never delete a source that was not verified.
- A moved directory with skipped conflicts keeps the skipped source files in place.

## 9. Copy integrity

Every file is written as: temp file in the destination directory (`.cpz-tmp-<random>`) → copy → apply metadata → optional fsync → optional verify → atomic rename → remove temp on any error.

- Source file changes size mid-copy: warn and mark the file failed; do not leave it in place.
- Verification honesty: read-back normally hits the page cache. Say in docs that `--verify` catches copy bugs and many I/O errors, but is not a guarantee against media corruption unless combined with `--fsync` and a cache drop. Do not claim more.
- `rename` gives atomic replacement, not crash durability. Durability needs `--fsync` including the parent directory.

## 10. Portability edge cases (must be handled and tested)

- Case-insensitive destinations (macOS default, Windows, some mounts): two sources differing only by case collide. Detect in planning.
- macOS Unicode normalization (NFD): treat normalization-equivalent names as colliding on such volumes.
- Windows: reserved names (`CON`, `NUL`, ...), trailing dots/spaces, `\\?\` long paths, symlink privilege errors with actionable messages.
- FAT/exFAT/network shares: coarse timestamps, no permissions, no symlinks. Warn when a requested preserve option cannot be honored; never claim full preservation.
- Filenames with spaces, quotes, leading dashes, newlines, Unicode, non-UTF-8 bytes.

## 11. Exit codes

| Code | Meaning |
|---|---|
| 0 | complete success |
| 1 | fatal error |
| 2 | usage or configuration error |
| 3 | partial failure: some files failed, others succeeded |
| 4 | unresolved conflicts (non-interactive, no policy) |
| 5 | verification mismatch |
| 130 | interrupted by SIGINT |
| 143 | terminated by SIGTERM |

When several apply, the highest-severity wins: 5 over 3 over 4 over 0. Skips chosen by policy are not failures.

## 12. Cancellation

Ctrl+C sets an atomic flag. Workers check between chunks and before starting a file, remove their temp files, flush any journal, print a short "cancelled" summary, exit 130. Abrupt kills (SIGKILL, power loss) can leave `.cpz-tmp-*` files: on the next run into the same destination, report stale temp files older than the current job and offer to remove them (never silently delete files you did not create in this job unless they match the temp pattern and `--clean-temp` is given).

## 13. Journal and resume (later phase)

Record canonical roots, options, policy, planned/completed/failed operations, source fingerprints (size, mtime, optional hash), schema version. Append-only records with checksums. Corrupt or truncated journal → clear error, never a panic. Paths read from a journal are untrusted: reject `..` traversal and paths escaping the recorded roots. Resume re-copies any file whose source fingerprint changed and any partially written file.

## 14. Sync (later phase)

`sync` never deletes without `--delete`. `--dry-run` lists every copy, update, skip, and delete. Excluded destination files are never deleted. Destination symlinks are unlinked, never followed, when deleting. The source is never modified. Create and copy before delete. Plan is validated in full before the first deletion.
