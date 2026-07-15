# eza-community/eza

> A modern `ls` replacement in Rust — the community-maintained continuation of the abandoned `exa`.

[GitHub repo](https://github.com/eza-community/eza) ·
[Official website](https://eza.rocks) ·
[License: EUPL-1.2](https://github.com/eza-community/eza/blob/main/LICENCE)

## Overview

eza is a single-binary command-line file lister, meant as a drop-in-ish
replacement for the Unix `ls`. It adds colour by file type and metadata,
Git status columns, icons (with Nerd Fonts), tree views, extended attributes,
and human-readable relative dates — features `ls` either lacks or exposes
awkwardly. It is written in Rust and distributes as one static-ish binary with
no runtime dependencies.

eza is a hard fork of `exa`, Benjamin Sago's original Rust `ls` replacement,
which had gone effectively unmaintained (its last release and largely dormant
issue tracker predate the fork)[^1]. The fork was started in 2023 by Christina
Sørensen and is now maintained by a small community organization rather than a
single author — a deliberate response to the bus-factor problem that killed
exa. Practically all downstream packagers (Homebrew, Arch, Nixpkgs, Debian)
have migrated their `exa` recipes to eza.

The defining tension is scope. eza's options are, in its own words, "almost, but
not quite, entirely unlike `ls`'s": it is not a byte-for-byte flag-compatible
replacement, so scripts written against `ls` output or flags will not blindly
work. It targets interactive human use — the shell alias, not the pipeline — and
that framing explains most of its design choices.

## Getting Started

```bash
# macOS / Linuxbrew
brew install eza
# Arch
pacman -S eza
# cargo (any platform with a Rust toolchain)
cargo install eza
# Nix flakes, no install
nix run github:eza-community/eza
```

```bash
# Common interactive setup — alias ls to a long, git-aware, icon'd listing
alias ls='eza --group-directories-first'
alias ll='eza -l --git --icons --header'
alias lt='eza --tree --level=2 --icons'

ll        # long view with Git status column, icons, and a header row
eza -laa  # long, all files including . and ..
```

## Architecture / How It Works

eza is a batch renderer, not a streaming one. It reads the target directory
entries, `stat`s each, optionally queries Git, then computes a full table
layout (column widths, grid packing) before printing anything. This is why the
grid and `--long` views align perfectly — but also why it holds the whole
listing in memory and why very large directories feel less instant than `ls`,
which streams.

Notable internals:

- **Git integration** uses libgit2 (via the `git2` crate) to read repository
  status per entry. The `--git` column and `--git-repos` mode open and query
  the repo, which is the single largest per-call cost; `--no-git` (or the
  faster `--git-repos-no-status`) exists specifically to opt out.
- **Icons** are Unicode Private Use Area glyphs from Nerd Fonts, mapped by file
  extension and name. They render as tofu boxes unless a patched Nerd Font is
  installed in the terminal — a frequent first-run confusion.
- **Colour and theming** started as `LS_COLORS` / `EXA_COLORS` environment
  variable compatibility and later gained a `theme.yml` config file
  (`$XDG_CONFIG_HOME/eza` or `$EZA_CONFIG_DIR`) covering both colours and icon
  overrides. Environment variables still win for backwards compatibility.
- **Hyperlinks, mount details, SELinux context, extended attributes** are all
  platform-conditional columns; several are Linux/macOS-only.

The codebase is a moderately sized Rust binary crate (options parsing, an
`fs` layer that models files and their metadata, and an output/rendering layer).
The EUPL-1.2 licence[^2] is unusual for the ecosystem — a European copyleft
licence with explicit compatibility bridges to GPL-family licences — and worth
noting for anyone who wants to vendor or relicense the code.

## Production Notes

**Do not treat it as `ls` in scripts.** Flag semantics and output differ. If a
script parses columns or depends on `ls` exit/format behaviour, keep `ls` (or
`command ls`) there and reserve eza for interactive aliases. This is the most
common real-world footgun.

**Git status is not free.** On large monorepos or directories inside big
repositories, the per-entry Git query dominates runtime. If `ll` feels slow,
the cause is almost always `--git`; drop it or use `--git-repos-no-status`.

**Icons require font setup.** Boxes/question marks mean the terminal font is not
a Nerd Font, not an eza bug. `--icons=never` (or unsetting the alias) is the
fallback; there is no way for eza to render glyphs the font does not contain.

**Not flag-stable with exa.** Aliases and dotfiles carried over from exa mostly
work, but a few flags were renamed or changed defaults during the fork. Re-read
`man eza` rather than assuming exa muscle memory transfers exactly.

**Windows support is real but secondary.** eza runs on Windows, but the metadata
model (permissions, owners, mounts, SELinux, xattrs) is Unix-centric; several
columns are no-ops or unavailable there.

**Colour-scale and relative-date output is for humans.** `--color-scale`,
`--time-style=relative`, and similar are tuned for eyeballing, not machine
parsing — another reason to keep it out of pipelines.

## When to Use / When Not

**Use when:**
- You want a nicer interactive directory listing: colours, icons, tree, Git
  status, sane human-readable sizes and dates.
- You already alias `ls`/`ll`/`lt` and want richer defaults with one binary.
- You liked exa and want a version that is still maintained and packaged.

**Avoid when:**
- You need strict `ls`/POSIX flag and output compatibility for scripts.
- You're on a minimal or embedded system where a Rust binary and libgit2 are
  unwanted weight versus busybox/coreutils `ls`.
- You want maximal throughput on enormous directories where streaming `ls` (or
  `fd`/`find` piped onward) beats a full-table renderer.

## Alternatives

- ogham/exa — the original this forks from; effectively unmaintained, so use eza instead unless you specifically need the historical binary.
- Peltoche/lsd — the other major Rust `ls` replacement with icons and colours; use it when you prefer its config/theming or defaults.
- sharkdp/fd — not a lister but a `find` replacement; use it when you're searching/filtering files to feed a pipeline rather than displaying a directory.
- bootandy/dust — a `du` replacement; use it when the question is "what's taking up space," not "what's here."
- coreutils/coreutils — the Rust reimplementation of GNU coreutils; use its `ls` when you need drop-in `ls` compatibility with a Rust toolchain.

## History

| Version | Date | Notes |
|---------|------|-------|
| exa (upstream) | 2015–~2021 | Benjamin Sago's original Rust `ls` replacement; later dormant[^1]. |
| eza fork | 2023-08 | Community fork started by Christina Sørensen after exa stalled[^3]. |
| 0.18.13 | 2024-04-25 | Referenced in README as a stable release line; man pages shipped in-repo[^4]. |
| theme.yml | 2024 | Config-file theming for colours and icons added atop env-var compatibility. |

## References

[^1]: exa upstream repository (unmaintained), ogham/exa. https://github.com/ogham/exa
[^2]: European Union Public Licence v1.2 (SPDX: EUPL-1.2). https://spdx.org/licenses/EUPL-1.2.html
[^3]: eza project site and community organization. https://eza.rocks
[^4]: eza CHANGELOG, entry `[0.18.13] - 2024-04-25`. https://github.com/eza-community/eza/blob/main/CHANGELOG.md

## Tags

rust, cli, ls-replacement, terminal, file-manager, coreutils, git, icons, command-line, developer-tools
