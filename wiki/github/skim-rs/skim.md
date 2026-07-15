# skim-rs/skim

> A fuzzy finder for the terminal, written in Rust, that ships both as the `sk` command and as an embeddable library crate.

[GitHub repo](https://github.com/skim-rs/skim) ·
[Official website](https://crates.io/crates/skim) ·
[License: MIT](https://github.com/skim-rs/skim/blob/master/LICENSE)

## Overview

skim is an interactive fuzzy finder: pipe it a list of lines (files, git branches, processes, history entries) and it narrows the list as you type, letting you pick one or many. It was created in 2016 by Jinzhou Zhang (lotabout) as a Rust answer to junegunn's Go-based `fzf`, and maintenance has since moved to the community-run `skim-rs` GitHub organization. It borrows `fzf`'s query syntax and much of its UX, so most of what you know from `fzf` transfers directly.

The defining tension for skim is that it lives in `fzf`'s shadow. `fzf` is one of the most widely deployed developer tools in existence, with a far larger user base, more shell/editor integrations, and more active feature development. skim is smaller and slower-moving. Its genuine reasons to exist are two: it is a single static Rust binary with no runtime, and — unusually for a fuzzy finder — it exposes a real library API, so Rust programs can embed the picker instead of shelling out to a separate process. If you don't need the library and you already have `fzf`, the case for switching is weak; if you want an embeddable Rust picker, skim is close to the only mature option.

skim's other distinguishing feature is **interactive mode** (`-i` / `-c`): rather than filtering a fixed input, skim re-runs an external command on every keystroke, feeding the query in as `{q}`. This makes it a live front-end for `grep`, `rg`, `ag`, or any command that takes a search term, which is a different workflow model from `fzf`'s default filter-a-stream behavior.

## Getting Started

```bash
# macOS
brew install sk
# Arch
pacman -S skim
# From crates.io (installs the `sk` binary)
cargo install skim
```

```bash
# Filter a stream: fuzzy-pick a file and open it in vim (-m = multi-select with TAB)
vim "$(find . -name '*.rs' | sk -m)"

# Interactive mode: re-run ripgrep on every keystroke, previewing matches live
sk --ansi -i -c 'rg --color=always --line-number {q}'
```

Shell key-bindings (Ctrl-T for files, Ctrl-R for history, Alt-C to `cd`) come from the scripts in `shell/`; source `key-bindings.{bash,zsh,fish}` to enable them.

## Architecture / How It Works

skim is a Rust workspace split into a **library crate** (`skim`) and a **binary** (`sk`) that is a thin CLI wrapper over the library. This split is the architectural point: the binary is just one consumer of an API that any Rust program can call. Input flows through a `SkimItemReader` that turns a byte stream into matchable items, an options builder configures the picker, and `Skim::run_with(...)` runs the interactive loop and returns the selected items. Because items are a trait, a host program can supply richer objects (with custom display and preview text) rather than plain lines.

Rendering historically used tuikit, lotabout's own terminal-UI library. The current codebase renders through **Ratatui** (the maintained successor to tui-rs), which is the mainstream Rust TUI stack — a meaningful reduction in bespoke, single-maintainer infrastructure.[^1]

Matching is pluggable across several scoring algorithms:[^2]

- **`skim_v2`** — the default, loosely modeled on `fzf`'s algorithm.
- **`fzy`** — a port of jhawthorn's `fzy` scoring, extended for basic typo tolerance.
- **`frizbee`** — the typo-resistant matcher from the `blink.cmp` Neovim completion plugin, pulled in as the `frizbee` crate.
- **`arinae`** — skim's own newer algorithm aimed at making typo-resistant matching feel natural without sacrificing per-item speed.

Matching runs across multiple threads, and results stream in as the source produces them, so skim stays responsive while a slow producer (like `find` over a large tree) is still emitting lines. Query parsing implements `fzf`-style tokens: `^prefix`, `suffix$`, `'exact`, `!inverse`, space as AND, ` | ` as OR, plus an optional `--regex` mode toggled at runtime with Ctrl-R.

## Production Notes

**It is not a 100% drop-in for `fzf`.** skim deliberately mirrors `fzf`'s syntax and many flags, but the flag sets have diverged over the years. A script or dotfile written against a recent `fzf` may reference options skim doesn't implement (or implements differently), so treat `sk` as "`fzf`-like," not "`fzf`-compatible," and test integrations rather than assuming parity. The README maintains an explicit "Differences from fzf" section for this reason.

**Library API stability.** If you depend on the `skim` crate, expect churn. The public types (options builder, item reader, item trait) have changed shape across releases, and upgrades have required code edits, not just a version bump. Pin the version and read the changelog before upgrading. This is the cost of skim being one of the few embeddable pickers — the API is still evolving.

**Windows support is second-class.** skim targets Unix terminals first; the repo carries a dedicated "Windows compatibility testing" note and troubleshooting entries for line-feed handling on Nix, FreeBSD, and termux. On Windows, `fzf` is the safer choice.

**Interactive-mode literalness.** In `-i -c '... {q}'` mode, `{q}` is substituted as the literal query wrapped in single quotes — the external command does an exact search, not a fuzzy one. Fuzzy narrowing only happens over a *piped* stream. Mixing the two mental models is a common source of "why isn't this fuzzy" confusion.

**Ecosystem gravity.** Editor and tool integrations overwhelmingly target `fzf` first. skim rides on projects that explicitly add a skim profile (for example the `fzf-lua` Neovim plugin, `nu_plugin_skim` for Nushell, or a SQLite scoring extension). Outside those, you may be the one writing the integration.

## When to Use / When Not

**Use when:**
- You want to embed fuzzy selection *inside* a Rust program via a library API rather than shelling out.
- You prefer a single dependency-free Rust binary and a from-source `cargo install` path.
- You want live-reload interactive mode as a front-end for `rg`/`grep`/`ag`.
- You're already on a distro that packages `skim` and want a `fzf`-like tool without installing Go tooling.

**Avoid when:**
- You just want the best-supported terminal fuzzy finder — `fzf` has more integrations, more maintainers, and faster feature cadence.
- You're on Windows and want a well-trodden path.
- You depend on a specific recent `fzf` flag or plugin; parity is not guaranteed.
- You need long-term API stability from an embedded picker and can't absorb breaking changes.

## Alternatives

- junegunn/fzf — the dominant fuzzy finder (Go); use it unless you specifically need a Rust library or want to avoid the Go binary.
- helix-editor/nucleo — a Rust fuzzy-matching *library* (no TUI); use it when you only need the matching algorithm to embed, not the interactive picker.
- jhawthorn/fzy — minimal, fast fuzzy finder in C; use it when you want something small and Unix-simple with no library ambitions.
- saghen/frizbee — the standalone typo-resistant matcher crate skim can use; use it directly when you want that algorithm without skim's UI.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-05 | Created by Jinzhou Zhang (lotabout) as a Rust fuzzy finder.[^3] |
| — | 2016–2023 | `0.x` line under lotabout; `fzf`-compatible syntax, interactive mode, library API. |
| — | ~2023–2024 | Maintenance transitioned to the community `skim-rs` organization. |
| — | recent | Rendering migrated to Ratatui; added `frizbee`, `fzy`, and in-house `arinae` matchers alongside default `skim_v2`.[^2] |

## References

[^1]: skim README — "Built with Ratatui" badge and rendering stack. https://github.com/skim-rs/skim
[^2]: skim README, "Algorithms" section (skim_v2, frizbee, fzy, arinae). https://github.com/skim-rs/skim#algorithms
[^3]: skim on crates.io / repository history — original author Jinzhou Zhang (lotabout), created 2016. https://crates.io/crates/skim

## Tags

rust, fuzzy-finder, cli, terminal, tui, fzf-alternative, ratatui, developer-tools, interactive, library
