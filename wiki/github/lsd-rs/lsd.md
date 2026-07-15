# lsd-rs/lsd

> A rewrite of GNU `ls` in Rust that adds colors, Nerd Font icons, a tree view, and richer formatting.

[GitHub repo](https://github.com/lsd-rs/lsd) ·
[License: Apache-2.0](https://github.com/lsd-rs/lsd/blob/master/LICENSE)

## Overview

`lsd` (LSDeluxe) is a drop-in-ish replacement for the `ls` command that leans
into terminal aesthetics: per-type colors, file-type icons drawn from patched
Nerd Fonts, a `--tree` view, human-readable sizes, and configurable date and
permission formatting[^1]. It was directly inspired by the Ruby project
`colorls`, and reimplements the idea as a single statically-linked Rust binary
so it installs cleanly across Linux, macOS, Windows, the BSDs, and Termux[^1].
The project started under the author `Peltoche` in 2018 and later moved to the
`lsd-rs` GitHub organization, which is why `github.com/Peltoche/lsd` links
redirect to the current path.

It sits in the "modern Unix CLI" family alongside `bat` (a `cat` replacement),
`fd` (a `find` replacement), and `eza` (another `ls` replacement). Its defining
tension is that of any beautified core-utility clone: the polish that makes it
appealing — icons, color themes, alignment — depends on the user's terminal and
font being configured correctly, and `lsd` is not, and does not aim to be, a
byte-for-byte POSIX-compatible `ls`. Scripts that parse `ls` output should not
be pointed at `lsd`; it is a human-facing interactive tool. The project is
actively maintained (last push mid-2026) with a large user base (~16k stars,
~500 forks)[^2], though its cadence is that of a mature utility — incremental
releases rather than rapid feature churn.

## Getting Started

Install from a package manager (preferred, since it also wires up man pages and
completions):

```sh
brew install lsd        # macOS / Linuxbrew
pacman -S lsd           # Arch
dnf install lsd         # Fedora
apt install lsd         # Debian bookworm+/Ubuntu 23.04+
scoop install lsd       # Windows
```

Or build from source with a Rust toolchain:

```sh
cargo install lsd
```

Icons require a patched **Nerd Font** installed and selected in your terminal;
without one you get boxes or question marks instead of glyphs[^1]. Typical
shell aliases:

```sh
alias ls='lsd'
alias ll='lsd -l'
alias la='lsd -a'
alias lt='lsd --tree'
```

## Architecture / How It Works

`lsd` is a single Rust binary built on `clap` for argument parsing. At a high
level it (1) resolves the flag set from CLI args merged with an optional
`config.yaml`, (2) walks the requested paths, `stat`-ing entries and collecting
metadata (permissions, owner/group, size, timestamps, symlink targets), (3)
maps each entry to an icon and color via its theme tables, and (4) lays the
result out into either a grid, a long listing, or a recursive tree.

Three layers of user customization stack on top of built-in defaults, each an
optional YAML file in the config directory[^1]:

- **`config.yaml`** — default flags (sorting, blocks shown in `-l`, date
  format, icon/color enablement).
- **`colors.yaml`** — overrides the color assigned to each file class; also
  honors the system `LS_COLORS` environment variable, including on Windows.
- **`icons.yaml`** — overrides icons by exact `name`, by `filetype`, or by file
  `extension`. Both Nerd Font glyphs and plain Unicode emoji are accepted, and
  the final set is a merge of your entries over the compiled-in defaults
  (`src/theme/icon.rs`).

Config file lookup follows the XDG Base Directory spec on Unix
(`$XDG_CONFIG_HOME/lsd` or `~/.config/lsd`) and checks
`%USERPROFILE%\.config\lsd` then `%APPDATA%\lsd` on Windows[^1]. Missing files
fall back to defaults silently, so a partial config (icons only, say) is valid.

The icon set is coupled to Nerd Font code-point layout, which is the main
external dependency that bites users. Nerd Fonts v3.0 relocated the Material
Design Icons range, so `lsd` shipped an updated icon set; fonts patched with an
older Nerd Fonts version render some glyphs wrong until re-patched[^1].

## Production Notes

- **Not a POSIX `ls`.** Output columns, coloring, and defaults differ from GNU
  `ls`. Do not use `lsd` in scripts that parse listing output. Alias it for
  interactive use, not for pipelines.
- **Font/terminal is the top support burden.** The single most common issue is
  missing or wrongly rendered icons, which is almost always an
  unpatched-font or terminal-config problem rather than an `lsd` bug. Verify
  with `echo $''` — a box or `?` means the font isn't set up[^1].
- **Terminal quirks.** Some emulators (Konsole, PuTTY/KiTTY on Windows) mangle
  wide icons or trim the first character of names; the README documents
  per-terminal workarounds and recommends Alacritty/Kitty as known-good[^1]. A
  `lsd --icon never --ignore-config` invocation isolates whether a rendering
  issue is icons or `lsd` itself.
- **Debian/Ubuntu `.deb` compression.** From `v1.1.0`, use the `_xz.deb`
  release on older distros; the default `.deb` uses zstd compression that only
  Debian 12 / Ubuntu 21.10 and newer can unpack[^3].
- **Icon-heavy output over slow links.** Full color + icon rendering emits many
  escape sequences and multibyte glyphs per line; over high-latency SSH or in
  huge directories this is heavier than plain `ls`. Disable per invocation with
  `--icon never` / `--color never`. Invalid UTF-8 filenames render with the
  `U+FFFD` replacement character rather than failing[^1].

## When to Use / When Not

**Use when:**
- You want a nicer interactive directory listing with colors, icons, and a tree
  view, and you control your terminal and fonts.
- You want one static binary that behaves consistently across Linux, macOS,
  Windows, BSD, and Termux.
- You already run the modern-CLI stack (`bat`, `fd`) and want a matching `ls`.

**Avoid when:**
- You need POSIX `ls` semantics or machine-parseable output for scripts.
- You're on a locked-down environment where installing a Nerd Font isn't
  possible — the headline feature degrades to plain text.
- You want the broadest feature set / most active development in this niche —
  `eza` currently iterates faster (git integration, hyperlinks, extended
  attributes).

## Alternatives

- eza-community/eza — actively developed `ls` replacement (successor to the
  abandoned `exa`); prefer it if you want git status columns and faster feature
  development.
- ogham/exa — the original Rust `ls` replacement; unmaintained, use eza instead.
- sharkdp/bat — a `cat` replacement with syntax highlighting; complementary, not
  a listing tool.
- sharkdp/fd — a `find` replacement; pairs with `lsd` in the same toolkit.
- athityakumar/colorls — the Ruby project that inspired `lsd`; use it if you
  live in a Ruby environment and don't want a compiled binary.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-11 | Repo created by `Peltoche`; early Rust rewrite of `ls`[^2]. |
| 0.x | 2019–2023 | Long 0.x series: icons, tree view, themes, config files. |
| 1.0.0 | 2023 | First 1.0 release; project under the `lsd-rs` org. |
| 1.1.0 | — | `_xz.deb` release variant added for older Debian/Ubuntu[^3]. |
| 1.2.0 | — | Current newest release per project README[^1]. |

## References

[^1]: lsd README — installation, configuration, theming, and troubleshooting. https://github.com/lsd-rs/lsd/blob/master/README.md
[^2]: lsd-rs/lsd repository metadata (stars, forks, license, dates) via GitHub API, retrieved 2026-07. https://github.com/lsd-rs/lsd
[^3]: lsd issue #891 — zstd `.deb` compression unsupported on older Debian/Ubuntu. https://github.com/lsd-rs/lsd/issues/891

## Tags

rust, cli, ls-replacement, terminal, coreutils, file-listing, nerd-fonts, icons, command-line, developer-tools
