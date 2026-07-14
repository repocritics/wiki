# nushell/nushell

> A shell whose pipelines carry structured data — tables and records — instead of raw text.

[GitHub repo](https://github.com/nushell/nushell) ·
[Official website](https://www.nushell.sh/) ·
[License: MIT](https://github.com/nushell/nushell/blob/main/LICENSE)

## Overview

Nushell (`nu`) is a shell and shell language built in Rust, first announced in
2019 by Jonathan Turner and collaborators[^1]. Its central bet is that a shell
pipeline should move *structured* values — tables, records, lists — rather than
byte streams of text. `ls` returns a table with typed columns; `ps` returns a
table of processes; `open Cargo.toml` parses the file into a record you can drill
into with `get package.version`. The recurring Unix ritual of `awk`/`cut`/`sed`
to re-parse text that some upstream command already had structured is, in Nu,
replaced by column access.

The defining tension is that this design puts Nu *outside* the POSIX shell
family. It is not `bash`-compatible and does not aim to be: scripts, quoting
rules, control flow, and even `&&`/`||` semantics differ. Nu is a new language
that happens to be a shell, closer in spirit to PowerShell and to functional
data tools than to `sh`. That buys consistency and typed data but costs
portability — you cannot paste most Stack Overflow shell answers and expect them
to run.

The second tension is stability. As of 2026 Nu is still on a `0.x` version line
and ships roughly monthly releases that regularly include intentional breaking
changes to command names, signatures, and language syntax[^2]. Many people run
it as a daily driver, but the project itself describes its design as still
subject to change. Treating a Nu config or script as a long-lived, upgrade-safe
artifact is not yet realistic.

## Getting Started

```bash
# Homebrew (Linux/macOS)
brew install nushell
# Windows
winget install nushell
# From source via crates.io
cargo install nu --locked
```

```nu
# Structured pipelines: filter a table by a typed column, no text parsing.
ls | where type == dir and size > 1mb | sort-by modified | select name size

# Load a file as data, drill in with column access.
open Cargo.toml | get package.version

# HTTP + JSON is first-class structured data.
http get https://api.github.com/repos/nushell/nushell | get stargazers_count
```

Running `nu` drops you into an interactive session backed by the Reedline line
editor. Config lives in `config.nu` (and historically `env.nu`); `$nu.config-path`
prints its location.

## Architecture / How It Works

Nu is a workspace of Rust crates (`nu-parser`, `nu-engine`, `nu-protocol`,
`nu-command`, `nu-cli`, and many more). A pipeline is parsed into an AST, type-
and signature-checked against known command definitions at parse time, then
evaluated by a tree-walking engine over an internal `Value` type. Because
commands declare typed signatures up front, Nu can flag unknown flags and arity
errors before running anything — closer to a compiled language's front end than
to a traditional shell's late string evaluation. The current engine and parser
are the result of the **engine-q** rewrite, which replaced the original
implementation and landed on `main` around the 0.60 release in early 2022[^3].

Every command falls into one of three pipeline roles: it produces a stream
(`ls`), transforms a stream (`where`, `select`, `sort-by`), or consumes one
(`table`, which is implicitly appended when output goes to a terminal). Values
flow lazily where possible so that large listings stream rather than fully
materialize.

**Plugins** extend Nu with out-of-process binaries named `nu_plugin_*` that
speak a defined protocol over stdio (JSON or MessagePack encoding). A plugin
registers its command signatures with the host and then exchanges structured
values the same way built-ins do — a filter plugin receives elements and streams
results back. The DataFrame support (`nu_plugin_polars`, built on the Polars
engine) ships this way, giving Nu columnar/Arrow-backed operations for large
tabular data separate from the default row-oriented `Value` model.

Interactivity is handled by **Reedline**, a from-scratch Rust line editor
maintained under the same org, which provides history, completions, hinting, and
multi-line editing. It replaced the earlier `rustyline` dependency.

## Production Notes

- **Not POSIX, by design.** You cannot use Nu as a drop-in replacement for `sh`
  in system scripts, CI `run:` steps that assume bash, or `#!/bin/sh` shebangs.
  External tools that generate shell snippets (installers, `eval "$(tool init)"`)
  usually need a Nu-specific integration path or will not work at all.
- **Breaking changes are routine.** Monthly releases have removed or renamed
  commands, changed operator behavior, and altered config format between
  versions[^2]. Pin the version you tested against, read the release notes before
  upgrading, and expect to touch your config on `0.x` bumps. The consolidation
  and reshaping of `env.nu`/`config.nu` over time is a recurring source of
  migration friction.
- **Startup cost and config.** A heavily customized config with many completions
  and third-party tool hooks (starship, zoxide, atuin) measurably slows shell
  startup; keep the hot path lean if you open many short-lived shells.
- **Structured parsing has edges.** `open`'s format autodetection and the
  `from *` / `to *` converters are convenient but not infallible on malformed or
  unusual files; verify parsing on data you do not control rather than assuming
  round-trip fidelity.
- **Ecosystem coverage.** Prompt tools, `zoxide`, `atuin`, `direnv`, `oh-my-posh`,
  and others ship official Nu integrations[^4], but the long tail of shell
  tooling assumes bash/zsh. Budget time to port dotfiles rather than reuse them.
- **Scripting maturity.** Error handling, module system, and closures exist and
  work, but the language is younger than its interactive polish; complex scripts
  are more likely to hit rough edges than one-liners.

## When to Use / When Not

**Use when:**
- You do a lot of interactive data wrangling — logs, CSV/JSON/TOML, process and
  filesystem tables — and want typed columns instead of text munging.
- You work cross-platform (Windows, macOS, Linux) and want one consistent shell
  language everywhere[^4].
- You value a coherent, discoverable command language over decades of `sh` lore.

**Avoid when:**
- You need POSIX compatibility, portable scripts, or drop-in bash replacement.
- You depend on a large corpus of existing shell scripts and tool init snippets.
- You need a shell whose language and config are frozen; `0.x` churn will bite.
- Your team can't absorb the learning curve of a genuinely different language.

## Alternatives

- PowerShell/PowerShell — the mature structured-object shell; use it when you're
  in the .NET/Windows ecosystem and want an established, stable object pipeline.
- fish-shell/fish — pick it when you want a friendlier *interactive* Unix shell
  without leaving the POSIX-adjacent world or learning a new data model.
- elves/elvish — a Go-based shell with structured data and a real scripting
  language; the closest philosophical sibling if Nu's churn worries you.
- oils-for-unix/oils — choose YSH/Oil when you want an *upgrade path from bash*
  that stays POSIX-compatible rather than a clean break.
- xonsh/xonsh — use it when you'd rather have Python as the shell language and
  keep seamless access to the Python ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2019-08 | Public announcement; text-vs-structured-data thesis, original engine[^1]. |
| ~0.60 | 2022-03 | engine-q rewrite merged to `main`: new parser/engine, parse-time checks[^3]. |
| — | 2022 | Reedline replaces rustyline as the interactive line editor. |
| 0.x | ongoing | ~monthly releases, frequent intentional breaking changes; no 1.0 as of 2026[^2]. |

## References

[^1]: Jonathan Turner, "Nushell" announcement (2019-08-23). https://www.jonathanturner.org/2019/08/introducing-nushell.html
[^2]: Nushell release notes / blog — per-release breaking-change sections. https://www.nushell.sh/blog/
[^3]: Nushell blog, "engine-q merged into main" (0.60, 2022). https://www.nushell.sh/blog/2022-02-22-nushell-0_60_0.html
[^4]: Nushell book — installation and platform support; official third-party integrations listed in the project README. https://www.nushell.sh/book/installation.html

## Tags

rust, shell, command-line, structured-data, pipeline, cross-platform, data-processing, terminal, scripting-language, cli
