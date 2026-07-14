# sharkdp/bat

> A `cat(1)` clone that adds syntax highlighting, Git change markers, and automatic paging — a terminal file viewer, not a text-stream tool.

[GitHub repo](https://github.com/sharkdp/bat) ·
[License: Apache-2.0 / MIT](https://github.com/sharkdp/bat/blob/master/LICENSE-APACHE)

## Overview

`bat` is a command-line file viewer written in Rust, first released in 2018[^1]. It reads files (or stdin) and prints them with syntax highlighting for a wide range of languages, a line-number sidebar, Git modification markers relative to the working tree, and a pager for output that exceeds one screen. It is one of the more widely adopted entries in the "modern Unix rewrite" cohort (alongside `ripgrep`, `fd`, `eza`), and is packaged in essentially every mainstream distribution.

The defining tension is that `bat` is often described as a drop-in `cat` replacement, but it is not one in the strict sense. When output goes to an interactive terminal it adds decorations and paging; when it detects a pipe or redirect it silently falls back to plain, `cat`-like behavior. That context-sensitivity is deliberate and usually convenient, but it means `bat` in a script behaves differently than `bat` at a prompt, which is the source of most surprises. `bat` is a viewer for humans reading code, not a byte-faithful concatenation tool for pipelines.

Despite eight years of production use and near-universal packaging, `bat` has never shipped a 1.0 release; it remains on the 0.x line (0.26.1 as of late 2025)[^2]. This reflects the maintainer's convention rather than instability — the tool is mature and its interface has been broadly stable — but it does mean the project reserves the right to make breaking changes across minor versions.

## Getting Started

```bash
# Debian/Ubuntu (note: binary may be named `batcat`, see Production Notes)
sudo apt install bat

# macOS / Linux via Homebrew
brew install bat

# from crates.io (requires Rust 1.79.0+)
cargo install --locked bat
```

```bash
# view a file with highlighting + line numbers
bat src/main.rs

# concatenate like cat (bat auto-detects the pipe and prints plain text)
bat header.md body.md footer.md > document.md

# force color/decorations when piping into another tool (e.g. fzf preview)
fzf --preview "bat --color=always --style=numbers --line-range=:500 {}"

# colorize man pages
export MANPAGER="bat -plman"
```

## Architecture / How It Works

`bat` is a thin orchestration layer over a small number of Rust libraries, most importantly `syntect`[^3], the same syntax-highlighting engine used by several editors. `syntect` parses Sublime Text `.sublime-syntax` grammars and `.tmTheme`/`.sublime-theme` color themes. `bat` bundles a large set of these grammars and themes and, at build time, compiles them into a single serialized binary cache that is embedded into the executable. This is why the `bat` binary is several megabytes rather than the tens of kilobytes of GNU `cat`, and why adding a custom syntax requires `bat cache --build` to regenerate the cache into your config directory.

Git integration is implemented with `git2` (libgit2 bindings). For each displayed file `bat` diffs the on-disk content against the Git index and renders added/modified/removed markers in the sidebar. This is a per-file diff against the index, not a full `git diff` renderer — it shows *that* lines changed, not the before/after content.

Paging is delegated, not built in. By default `bat` pipes its output to `less` (with flags that preserve color and disable line wrapping), controlled via `--paging`, `PAGER`, and `BAT_PAGER`. `bat` is not itself a pager; if `less` is absent or misconfigured, paging behavior degrades. Output-context detection is central: `bat` checks whether stdout is a TTY and switches between the decorated interactive path and the plain-text pipe path automatically, which is the mechanism behind its dual `cat`-like personality.

## Production Notes

**The `batcat` naming clash.** On Debian and older Ubuntu releases the executable is installed as `batcat`, because the name `bat` collides with an unrelated existing package[^4]. Scripts and docs that assume a `bat` binary will fail on these systems until users create a `bat -> batcat` symlink or alias. This is the single most common first-run confusion.

**Pipe behavior is a footgun in scripts.** Because `bat` strips decorations and color when it detects a non-interactive stdout, a command that looks colorized at the prompt produces plain text when captured in `$(...)`, piped, or redirected. To keep color/decorations in a pipeline you must pass `--color=always` and/or `--decorations=always` (or the `--force-colorization`/`-f` alias). Conversely, do not rely on `bat` for byte-exact concatenation in automation where a trailing-newline or decoration difference matters — use `cat`.

**Not free for large or streaming files.** Syntax highlighting has real per-line cost, and `bat` loads its syntax set on startup. For very large files, log tailing, or hot loops, `cat`/`--plain` is meaningfully faster. When following logs (`tail -f | bat`) you must pass `--paging=never` and usually an explicit `-l`/`--language`, since a paged or auto-detected stream will not behave as expected.

**Pager coupling.** Corrupted or missing color in the pager usually traces back to `less`: old `less` versions and incorrect `LESS`/`BAT_PAGER` settings are the usual culprits. On Windows there is no system `less`, so paging depends on what is on `PATH`. The Visual C++ Redistributable is a prerequisite for the prebuilt Windows binaries.

**Custom syntaxes/themes require a cache rebuild.** New `.sublime-syntax` or theme files placed in `bat --config-dir` are not picked up until `bat cache --build` runs; `bat cache --clear` reverts to bundled defaults. Theme selection is via `--theme`/`BAT_THEME`, with separate `--theme-dark`/`--theme-light` (and matching env vars) added in the 0.26 line for automatic light/dark switching.

## When to Use / When Not

**Use when:**
- You read source, configs, or logs at the terminal and want highlighting, line numbers, and Git markers without opening an editor.
- You want a better previewer for `fzf`, `find`/`fd`, or `ripgrep` results.
- You want a colorizing pager for `man`, `--help` output, or `git show`.

**Avoid when:**
- You need byte-faithful concatenation or maximum throughput in scripts and pipelines — use `cat`.
- You want rich side-by-side Git diffs — `bat`'s markers only indicate changed lines; use a dedicated diff pager.
- You are on a minimal/embedded system where a multi-megabyte binary and a `less` dependency are unwelcome.

## Alternatives

- dandavison/delta — use instead when your primary need is viewing `git diff`/`git blame` with syntax-highlighted, side-by-side output rather than viewing whole files.
- coreutils/coreutils (`cat`) — use for plain concatenation, scripting, and streaming where color and decorations are unwanted and startup cost matters.
- noborus/ov — use when you mainly want a modern pager (search, column mode) rather than a file viewer; can be paired with `bat` as its highlighter.
- walles/moar — use when you want a friendlier `less` replacement with built-in highlighting and simpler defaults.
- sharkdp/fd — not a `bat` substitute, but the companion file-finder from the same author frequently used together (`fd ... -X bat`).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-05 | Initial release; `syntect`-based highlighting, Git integration[^1]. |
| 0.10.0 | 2019-02 | Broad distro packaging era begins. |
| 0.13.0 | 2020-03 | Continued syntax/theme expansion. |
| 0.18.0 | 2021-02 | Widely referenced packaged release (`0.18.3` used in install docs). |
| 0.22.0 | 2022-09 | Feature and syntax updates. |
| 0.24.0 | 2023-10 | Maintenance and highlighting improvements. |
| 0.25.0 | 2025-01 | Continued 0.x maintenance line. |
| 0.26.0 | 2025-10 | Separate dark/light theme selection (`--theme-dark`/`--theme-light`). |
| 0.26.1 | 2025-12 | Latest release as of this writing[^2]. |

## References

[^1]: `sharkdp/bat` repository, created 2018-04-21. https://github.com/sharkdp/bat
[^2]: `bat` releases (v0.26.1, published 2025-12-02). https://github.com/sharkdp/bat/releases
[^3]: `syntect` — Rust syntax highlighting via Sublime Text grammars. https://github.com/trishume/syntect
[^4]: `bat` issue #982, executable name clash on Debian/Ubuntu. https://github.com/sharkdp/bat/issues/982

## Tags

rust, cli, command-line, syntax-highlighting, terminal, pager, git, cat-clone, developer-tools, file-viewer
