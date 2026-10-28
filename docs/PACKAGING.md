# PACKAGING: naming, licensing, GitHub, Debian

## 1. Choose the name first

`rcp` (the name used in the original draft) is a poor choice: it is the traditional BSD **remote copy** command (shipped in Debian as part of the `rsh-client` package, as far as I know), so it collides with an existing command and a Debian package would conflict. `cpz` is used here only as a **working name**.

Before settling, check all of these and pick something short, typeable, and unclaimed:
- `apt-file search -x 'bin/<name>$'` on Debian (and `apt search <name>`)
- crates.io, GitHub, and a general web search for existing tools
- `command -v <name>` on a few distros

A name that doesn't look like a drop-in for `cp` is fine; do **not** install a binary called `cp`.

## 2. License

Pick before the first public commit. MIT OR Apache-2.0 (the Rust convention) or GPL-2.0-or-later/GPL-3.0 are all acceptable to Debian main. Rules:
- Every dependency must be DFSG-free and license-compatible.
- Add `LICENSE-*` files, SPDX headers or `license` in `Cargo.toml`, and keep `cargo deny` (licenses + advisories) in CI.

## 3. GitHub project checklist

- README: what it is, a screenshot/asciinema of the progress UI, install, quick examples, an honest comparison with `rsync`, `advcpmv`, and other Rust copy tools, safety guarantees, and known limits.
- `CHANGELOG.md` (Keep a Changelog), semantic versioning, git tags.
- `CONTRIBUTING.md`, `SECURITY.md`, issue templates.
- CI: Linux/macOS/Windows build and test, clippy, fmt, `cargo deny`, `cargo audit`.
- Releases: prebuilt static binaries (`x86_64-unknown-linux-musl`, `aarch64`), checksums, signed tags. `cargo install` support. Optionally `cargo-deb` for a quick `.deb` for early adopters.
- Man page (`cpz.1`) and shell completions generated at build time or checked in.

## 4. Getting into Debian (be realistic)

**Goal A: available via `apt install` in the Debian archive: achievable.** It takes months and requires work, but ordinary software gets there routinely.

**Goal B: installed by default / replacing `cp`: not realistic, and you should not plan around it.** `cp` is part of GNU coreutils, which is an *Essential* package in Debian. Default-installed sets are decided by Debian policy and the installer/task maintainers, and are very conservative about adding new tools, let alone a brand-new one. A project that is not yet widely used will not be proposed for that. Aim for "easy to install", and let adoption decide the rest.

### Process, in order
1. **Release a stable 1.0-ish version** with tests, a license, a man page, and a changelog. Debian maintainers want an upstream that is maintained and releases tarballs/tags.
2. **Dependencies must be in Debian.** Debian builds offline from packaged source. Each Rust crate you depend on needs a Debian `librust-*-dev` package at an acceptable version (the Debian Rust team packages crates with `debcargo`). If a dependency is missing or too old, someone has to package it first. **This is the main reason to keep the dependency list short** (see ARCHITECTURE §8). Check `https://tracker.debian.org` / `apt search librust-<crate>-dev` for each crate before adopting it.
3. **File an ITP** ("Intent To Package") bug against the `wnpp` pseudo-package using `reportbug`.
4. **Package it** (`debian/` directory), ideally with the Debian Rust team workflow (`debcargo` / `dh-cargo`). Needs: `debian/control`, `copyright` (DEP-5), `changelog`, `rules`, man page, passes `lintian` cleanly.
5. **Find a sponsor.** You are not a Debian Developer, so you upload to **mentors.debian.net** and request sponsorship (mailing list `debian-mentors`, or the Debian Rust team). The sponsor reviews and uploads.
6. Package lands in **unstable (sid)**, migrates to testing, then appears in the next stable release (Debian releases roughly every two years; there are freeze periods when new packages are not accepted).
7. Ubuntu and derivatives typically sync from Debian afterward.

Other distributions are easier first steps and build credibility: AUR, Homebrew, a Nix package, `crates.io` (`cargo install`), Fedora COPR, and a Flatpak is not needed for a CLI.

### Requirements Debian will care about
- Reproducible, offline build from a source tarball with `Cargo.lock` handled per Debian practice.
- No network access or telemetry at runtime.
- No bundled vendored crates in the Debian package.
- A clear security story for a tool that touches user data: tests, no `unsafe`, a `SECURITY.md`.
- A responsive maintainer: Debian expects bugs to be answered.

## 5. Distribution of the tool name vs. coreutils

Never ship a file named `cp`, `mv`, or `rcp`. Users who want it as their default can alias it themselves. Document that in the README (`alias cp=cpz` is a personal choice, not a default; the flag semantics differ in places, see SPEC §3).
