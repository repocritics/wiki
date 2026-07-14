# sharkdp/fd

> A user-friendly, gitignore-aware alternative to `find` — opinionated defaults over `find`'s exhaustive flag surface.

[GitHub repo](https://github.com/sharkdp/fd) ·
[License: MIT OR Apache-2.0](https://github.com/sharkdp/fd/blob/master/LICENSE-MIT)

## Overview

`fd` is a command-line tool for finding filesystem entries by name, written in Rust by David Peter (sharkdp) and first released in 2017[^1]. It is explicitly not a full reimplementation of GNU `find`: it drops most of `find`'s predicate DSL in favor of a short, regex-first invocation (`fd PATTERN`) and a set of defaults chosen for interactive use. The tradeoff is deliberate — `fd` is faster to type and faster to run for the common case, but it will never be a drop-in replacement for `find`'s full expression language.

Two of those defaults define the tool's personality and cause most of its surprises. First, `fd` respects `.gitignore`, `.ignore`, and `.fdignore` files and skips hidden entries unless told otherwise, so results are scoped to "files you probably care about" rather than "every inode on the disk." Second, the search pattern is a regular expression matched against the filename only (not the full path) with smart-case behavior: case-insensitive until the pattern contains an uppercase letter. Users coming from `find` are regularly confused when `fd` "can't find" a file that is merely hidden or gitignored — the fix is `-H`/`-I` or `-u`/`--unrestricted`.

`fd` shares its performance-critical machinery with ripgrep: the `regex` and `ignore` crates (both by Andrew Gallant / BurntSushi) do the pattern matching and the parallel, gitignore-aware directory traversal[^2]. This lineage is why `fd`, `rg`, and similar Rust CLI tools behave consistently around ignore files and why their traversal speed is comparable.

## Getting Started

```bash
# Rust toolchain
cargo install fd-find          # crate is "fd-find"; binary is "fd"

# macOS
brew install fd

# Debian / Ubuntu — apt package is fd-find and installs the binary as `fdfind`
sudo apt install fd-find
```

```bash
# Find files/dirs whose name contains "config" under the current tree
fd config

# Regex: names starting with x, ending in rc
fd '^x.*rc$' /etc

# All Markdown files, then run a command per result (in parallel)
fd -e md -x wc -l

# Search everything, including hidden and gitignored entries
fd -u pattern

# Only files, exclude .git, feed an editor once with all results
fd -tf -E .git 'test_.*\.py' -X vim
```

## Architecture / How It Works

`fd` is a relatively thin, opinionated front-end over two libraries that do the heavy lifting:

- **Traversal** — the `ignore` crate walks the directory tree in parallel across a thread pool (default: number of logical cores), applying `.gitignore`/`.ignore`/`.fdignore` rules and hidden-file filtering as it descends. Pruning ignored directories early is a large part of why `fd` beats `find` on real project trees: it never recurses into `node_modules`, `target`, or `.git` unless asked.
- **Matching** — by default the pattern is compiled with the `regex` crate and tested against each entry's file name. `--glob`/`-g` switches to glob semantics; `--full-path`/`-p` matches against the whole path instead of just the basename.

Command execution (`-x`/`--exec` and `-X`/`--exec-batch`) is modeled on GNU Parallel's placeholder syntax: `{}` (full path), `{.}` (path without extension), `{/}` (basename), `{//}` (parent dir), `{/.}` (basename without extension). `-x` spawns one process per result across the same thread pool; `-X` collects all results and invokes the command once. `fd` guarantees that stdout/stderr from parallel `-x` children are not interleaved, which makes `fd -x` a serviceable lightweight `xargs -P` for file-oriented tasks.

Coloring reuses the `ls` model: file-type colors are driven by the `LS_COLORS` environment variable (and `fd` honors `NO_COLOR`). This means `fd` inherits whatever `dircolors` configuration the shell already has, rather than shipping its own theme.

## Production Notes

- **The binary is `fdfind` on Debian/Ubuntu.** The `fd` name collides with an existing package, so Debian and Ubuntu ship the binary as `fdfind`. Scripts written against `fd` break on these systems unless you symlink or alias (`ln -s $(which fdfind) ~/.local/bin/fd`). The crate on crates.io is likewise `fd-find`, not `fd`.
- **Ignore-aware by default is a footgun in automation.** In CI or cron, `fd` inside a repo silently omits gitignored and hidden paths. If a script must see everything, pass `-u`/`--unrestricted` (or `-HI`) explicitly; do not assume `fd pattern` and `find -name` return the same set.
- **Filename-only matching surprises `find` users.** `fd foo/bar` will not match a nested path the way you might expect — add `-p`/`--full-path` to match against the whole path. Similarly, patterns that begin with `-` need a `--` separator or a character-class trick (`fd -- -pattern`).
- **`-x` cannot call shell aliases or functions.** Command execution runs actual executables, not shell builtins, aliases, or functions. To use a function you must invoke a shell explicitly: `fd … -x bash -c 'my_fn "$1"' bash`.
- **`fd … -X rm -r` has race hazards on nested same-named directories.** Removing all directories named `foo` in a tree like `foo/bar/foo` can delete the outer directory before the inner path is processed, producing (harmless but noisy) "No such file or directory" errors. Dry-run without `-X rm` first.
- **Benchmarks are workload-dependent.** The README's headline (roughly an order of magnitude faster than `find -iregex` on a large home directory) reflects one machine and a warm disk cache[^3]. On cold caches or I/O-bound network filesystems the traversal parallelism helps less; measure your own workload with hyperfine rather than trusting the ratio.

## When to Use / When Not

**Use when:**
- You interactively search project trees and want gitignore-aware, hidden-skipping defaults.
- You want a short, regex-first syntax and parallel `-x`/`-X` execution over results.
- You are wiring a file-list source for `fzf`, `rofi`, or an editor's find-file.

**Avoid when:**
- You need `find`'s full predicate/expression language (complex `-and`/`-or` trees, `-printf` formatting, `-newer` combinations) — `fd` intentionally does not cover all of it.
- You need guaranteed POSIX portability with zero installs on a locked-down host; `find` is already there.
- You want an indexed, near-instant whole-system name lookup — a `locate`-style index fits better than a live traversal.

## Alternatives

- GNU findutils (`find`) — the baseline `fd` reworks; use it when you need the full expression DSL, `-printf`, or POSIX portability without installing anything.
- BurntSushi/ripgrep — content search, not filename search; `rg --files` also emits a gitignore-aware file list, so reach for it when you're searching inside files or already standardized on `rg`.
- junegunn/fzf — interactive fuzzy filter that complements rather than replaces `fd` (commonly fed by `fd`); use it when the goal is interactive selection.
- jhspetersson/fselect — SQL-like queries over the filesystem; use it when you need rich metadata predicates (size/date/attrs) expressed as queries.
- plocate / mlocate (`locate`) — prebuilt index for instant repeated name lookups; use it when a periodically stale index is acceptable and you want sub-millisecond results.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2018-01 | First 1.0; regex search, smart case, gitignore-aware defaults[^4]. |
| 7.0.0 | 2018-08 | `--exec`/`--exec-batch` and placeholder syntax matured. |
| 8.0.0 | 2020-01 | Defaults/flags cleanup; `--unrestricted` and ignore-file handling refined[^4]. |
| 9.0.0 | 2023-08 | Batch-size control for `-X`, filter and output improvements[^4]. |
| 10.0.0 | 2024-02 | `--format` template output; `--hyperlink`; assorted breaking-flag tidy-up[^4]. |

## References

[^1]: `sharkdp/fd` repository and README — "A simple, fast and user-friendly alternative to 'find'." https://github.com/sharkdp/fd
[^2]: `fd` README, benchmark section: credits the `regex` and `ignore` crates shared with ripgrep. https://github.com/BurntSushi/ripgrep
[^3]: `fd` benchmark scripts and methodology. https://github.com/sharkdp/fd-benchmarks
[^4]: `fd` releases and changelog. https://github.com/sharkdp/fd/releases

## Tags

rust, cli, command-line, file-search, find-alternative, filesystem, regex, gitignore, developer-tools, terminal, unix
