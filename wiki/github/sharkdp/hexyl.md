# sharkdp/hexyl

> A colored command-line hex viewer — a read-only `xxd`/`hexdump` replacement tuned for reading, not patching.

[GitHub repo](https://github.com/sharkdp/hexyl) ·
[License: Apache-2.0 OR MIT](https://github.com/sharkdp/hexyl/blob/master/LICENSE-APACHE)

## Overview

`hexyl` is a terminal hex viewer written in Rust by David Peter (sharkdp), first
published in late 2018[^1]. Its single differentiating idea is color: every byte
is classified into one of a handful of categories — NULL bytes, printable ASCII,
ASCII whitespace, other ASCII control bytes, and non-ASCII (`> 0x7F`) — and each
category gets its own color in both the hex panel and the character panel[^2].
The result is that structure in a binary (string tables, zero padding, high-bit
data) is visible at a glance rather than something you decode byte by byte from a
monochrome dump.

The defining tradeoff is that `hexyl` is a *viewer*, not an editor and not a
scripting primitive. It has no in-place editing, no "reverse" mode to turn a dump
back into bytes, and its default output carries Unicode box-drawing borders and
ANSI color that are meant for a human terminal rather than for `grep`/`diff`.
Where `xxd` doubles as a binary-patching tool (`xxd` then `xxd -r`) and `od`
emits stable, parseable columns, `hexyl` deliberately optimizes for the one task
of *looking* at bytes. It has never shipped a 1.0 and remains on a `0.x` line, so
it treats itself as a small, stable utility rather than a stabilized API.

Like the author's other tools (`fd`, `bat`, `hyperfine`), `hexyl` is a focused
single-purpose CLI with opinionated defaults; the binary and crate are both named
`hexyl`, and it is packaged in most mainstream distributions and package managers.

## Getting Started

```bash
cargo install hexyl        # Rust 1.56+; crate and binary are both "hexyl"
brew install hexyl         # macOS
sudo apt install hexyl     # Debian/Ubuntu (19.10+), Fedora, Arch, etc.
```

```bash
# Dump a file
hexyl /bin/ls

# First 64 bytes only, skipping the first 32
hexyl --length 64 --skip 32 firmware.bin

# Read from stdin
head -c 128 /dev/urandom | hexyl

# Two-panel default, or force a single panel and 4-byte groups
hexyl --panels 1 --group-size 4 data.bin

# Show byte values in binary instead of hex, plain ASCII borders
hexyl --base binary --border ascii data.bin
```

## Architecture / How It Works

`hexyl` streams bytes from a file or stdin and renders the classic three-region
hex layout: a left offset column, the hex byte values in the middle (16 bytes per
row, split into groups), and a character panel on the right. The middle values
can be rendered in binary, octal, decimal, or hexadecimal via `--base`, and the
row can be split across one or more panels (`--panels`) with a configurable byte
`--group-size`[^2].

The coloring is the core logic: each byte is mapped to a category, and the same
category color is used for that byte in both the numeric and character panels, so
your eye can correlate the two. Category colors are configurable through
`HEXYL_COLOR_*` environment variables (`HEXYL_COLOR_ASCII_PRINTABLE`,
`HEXYL_COLOR_NULL`, `HEXYL_COLOR_NONASCII`, `HEXYL_COLOR_OFFSET`, and others),
accepting the eight standard terminal colors, their "bright" variants, or `#rrggbb`
hex values[^2]. Coloring respects `--color <auto|always|never>`.

`hexyl` also collapses runs of identical rows — "squeezing" — replacing long
stretches of repeated bytes (typically zero padding) with a single marker line so
that a mostly-empty region does not scroll a screenful of identical output. This
is on by default and disabled with `--no-squeezing` / `-v`.

Beyond the binary, `hexyl` publishes a library crate exposing its `Printer` so
other Rust programs can produce the same colored dump programmatically rather than
shelling out. The tool is thin: argument parsing plus the printer, with no plugin
system, no config file, and no interactive mode.

## Production Notes

- **It is read-only.** There is no editing and no reverse/patch mode. If you need
  to *modify* bytes, `hexyl` is the wrong tool — use `xxd -r`, `dd`, or a real hex
  editor. `hexyl` shows you where to patch; something else does the patch.
- **Default output is for humans, not pipelines.** Borders and ANSI color make
  `hexyl | grep`, `hexyl | diff`, and byte-offset arithmetic awkward. For
  scripting, stable machine-readable dumps, or diffing two binaries, `od -A x -t
  x1z` or `xxd` are the better sources; reach for `hexyl` at the interactive
  prompt, not inside a script.
- **Squeezing hides data by design.** Because identical rows are collapsed by
  default, a naive `hexyl file` does not show every byte of a zero-padded region.
  When you need the literal full dump (e.g. verifying padding), pass `-v` /
  `--no-squeezing`.
- **No implicit paging or length cap.** Without `--length`, `hexyl` dumps the
  entire input; on a multi-megabyte file this floods the terminal. Combine with
  `--length`/`--skip` to window into a file, or pipe into a pager.
- **Windows needs an ANSI-capable terminal.** Color depends on ANSI escape
  support — Windows Terminal or ConHost v2 (Windows 10 1703+); older consoles show
  escape sequences or no color[^2].
- **Color config is environment-only.** There is no config file; persistent
  color choices must live in shell startup as `HEXYL_COLOR_*` exports, which is
  easy to forget across machines.

## When to Use / When Not

**Use when:**
- You are interactively inspecting a binary and want byte categories (null,
  printable, control, high-bit) visually separated.
- You want a quick, good-looking dump of a file or piped stream with sensible
  offset/hex/ASCII columns and no configuration.
- You want binary/octal/decimal value display or multi-panel layouts without
  memorizing `od` format strings.

**Avoid when:**
- You need to edit or patch bytes — use a hex editor or `xxd -r`.
- You need stable, parseable output for scripts, diffs, or byte-offset tooling —
  `od`/`xxd` are more predictable.
- You are on a locked-down host where installing a Rust binary is not worth it and
  `xxd`/`od`/`hexdump` are already present.

## Alternatives

- vim/vim — `xxd` ships with Vim; use it when you also need to convert a dump back
  into binary (`xxd -r`) or want a tool that is already installed nearly everywhere.
- coreutils/coreutils — `od` gives POSIX, script-stable octal/hex/character dumps;
  use it when output has to be parsed or reproduced deterministically.
- WerWolv/ImHex — a full graphical hex editor and reverse-engineering suite; use it
  for pattern languages, data inspection, and editing rather than terminal viewing.
- pixel/hexedit — a classic curses-based terminal hex *editor*; use it when you need
  to change bytes in place from a TTY.
- sharkdp/bat — sibling tool for *text* files (syntax highlighting, paging); reach
  for it when the input is source or logs, not raw binary.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-11 | Initial release: colored 16-byte hex + offset + character panels[^1]. |
| 0.x (feature line) | 2019–2025 | Added squeezing controls, `--base`, multi-`--panels`, `--group-size`, `--border` styles, and `HEXYL_COLOR_*` configuration[^3]. |
| 0.15.0 | recent | Current `0.x` release line at time of writing; project remains pre-1.0[^2]. |

## References

[^1]: `sharkdp/hexyl` repository — "A command-line hex viewer," created 2018-11-05. https://github.com/sharkdp/hexyl
[^2]: `hexyl` README — byte categories, `HEXYL_COLOR_*` configuration, installation, Windows ANSI requirement, dual MIT/Apache license. https://github.com/sharkdp/hexyl/blob/master/README.md
[^3]: `hexyl` releases and changelog. https://github.com/sharkdp/hexyl/releases

## Tags

rust, cli, command-line, hex-viewer, hexdump, binary-data, hexadecimal, developer-tools, terminal, unix, reverse-engineering
