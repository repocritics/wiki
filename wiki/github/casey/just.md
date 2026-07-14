# casey/just

> A command runner with `make`-like syntax that is deliberately not a build system.

[GitHub repo](https://github.com/casey/just) ·
[Official website](https://just.systems) ·
[License: CC0-1.0](https://github.com/casey/just/blob/master/LICENSE)

## Overview

`just` is a command runner written in Rust by Casey Rodarmor, first committed in
2016[^1]. Project-specific commands ("recipes") live in a file called `justfile`
whose syntax is intentionally close to a `Makefile` — colon-terminated recipe
names, tab/space-indented bodies, dependencies listed after the name — but the
resemblance ends at the surface. `just` does not track file modification times,
does not build a target dependency graph, and does not decide what is "up to
date." It runs the recipe you ask for, after running its dependencies, every
time. This is the defining design choice: it takes `make`'s ergonomic recipe
syntax and drops the build-system semantics that make `make` frustrating for
task-running[^2].

The audience is developers who want a single, discoverable place for the
scattered commands a project accumulates — `test`, `lint`, `deploy`, `migrate` —
without the `.PHONY` boilerplate, silent recipe failures, and shell-vs-make
variable confusion that plague `Makefile`s used as task runners. It ships as a
single dependency-free binary on Linux, macOS, Windows, and the BSDs. The
central tension is exactly that it is *not* a build system: if your problem
needs incremental compilation, file-timestamp dependency resolution, or parallel
target scheduling, `just` will not do it and is not trying to. Since 1.0 it
carries a strong backwards-compatibility commitment — there will never be a
`just` 2.0, and any incompatible change is opt-in per `justfile`[^3].

## Getting Started

```bash
cargo install just          # from source via crates.io
brew install just           # macOS / Linux
apt install just            # Debian 13 / Ubuntu 24.04+
winget install --id Casey.Just --exact   # Windows
```

Create a `justfile`:

```just
# variables use := , not =
project := "my-app"

# the first recipe is the default when you run `just` with no args
default:
    @just --list

# a recipe with a dependency and a parameter
test filter="":
    cargo test {{filter}}

build: test
    cargo build --release

# a shebang recipe runs as a single script, not line-by-line
deploy env:
    #!/usr/bin/env bash
    set -euo pipefail
    echo "deploying {{project}} to {{env}}"
    ./scripts/deploy.sh {{env}}
```

```console
$ just test integration
cargo test integration
$ just --list
Available recipes:
    build
    default
    deploy env
    test filter=""
```

## Architecture / How It Works

`just` is a single Rust binary with no runtime dependencies beyond a shell. On
invocation it locates the nearest `justfile` (searching upward from the current
directory, so recipes run from any subdirectory), lexes and parses it into an
AST, and performs static analysis *before executing anything*: unknown recipes,
undefined variables, and circular dependencies are reported up front rather than
mid-run[^2]. This is a deliberate contrast to shell scripts that fail halfway.

Execution model, and the part that surprises newcomers: each **line** of a
normal recipe is run as a separate invocation of the shell (default `sh -cu`).
State does not persist between lines — a `cd` on one line does not affect the
next. To run a recipe body as one connected script you use a **shebang recipe**
(a body beginning with `#!`), which `just` writes to a temporary file and
executes as a single program; this is how you get Python, Node, or `bash -euo
pipefail` recipes[^2].

The expression language is small but real: `:=` assignments, string
interpolation with `{{...}}`, conditionals, a library of built-in functions
(path manipulation, environment lookup, hashing, `os()`/`arch()`), and
`set`-based settings that configure the shell, dotenv behavior, positional
arguments, and more. Larger projects are split with `import` (textual inclusion)
and `mod` (submodules with their own `justfile`), both later additions to the
language.

What is intentionally absent: no file-target tracking, no `mtime` comparison, no
`--jobs` parallel scheduling, no incremental rebuild. Dependencies are run
unconditionally (once per invocation, deduplicated). If recipe `build` depends
on `test`, `test` always runs first — there is no "skip because nothing
changed." That logic is yours to add or to delegate to a real build tool called
*from* a recipe.

## Production Notes

**It is not a build system — do not fake one.** People reach for `just` and then
try to reconstruct `make`'s "rebuild only if source is newer" behavior. `just`
has no primitive for this. If you need it, call `make`, `ninja`, `cargo`, or a
language-native build tool from inside a `just` recipe and let `just` be the
top-level menu.

**Per-line shell invocation is the number-one footgun.** `cd build` followed by
`cmake ..` on the next line will not work — the directory change is gone. Fix it
with a single `&&` chain, `set working-directory`, or (cleanest for anything
non-trivial) a shebang recipe. Every team using `just` hits this at least once.

**Windows needs a shell.** By default recipes run under `sh`, which Windows does
not ship. You must install Git Bash / Cygwin with `sh` on `PATH`, or override
with `set shell := ["powershell.exe", "-c"]` (or `cmd.exe`). POSIX-shell recipes
are inherently not portable to a stock Windows machine; truly cross-platform
justfiles tend to use shebang recipes in a language installed everywhere.

**The npm package is `rust-just`, not `just`.** The `just` name on npm belongs
to an unrelated, older JavaScript project, so `npm install -g rust-just` (and
the same on pipx / uv) is the correct incantation[^4]. Installing `just` from
npm gets you the wrong software.

**Dotenv loading is opt-in and this changed historically.** Early versions
auto-loaded `.env`; current versions require `set dotenv-load := true`. Recipes
that silently relied on `.env` variables are a classic breakage when upgrading
across that boundary. Separately, the default `sh -cu` enables `-u`, so
referencing an unset shell variable is a hard error inside recipes — usually
desirable, but a surprise when porting existing scripts[^2].

**Upgrades are low-risk by policy.** Post-1.0, breaking changes to existing
justfiles are off the table, so version bumps are generally safe. Features not
yet stabilized are gated behind `--unstable` / `set unstable` and error by
default. You can pin a floor with `set minimum-version := '1.55.0'`.

## When to Use / When Not

**Use when:**
- You want a discoverable, self-documenting menu of project commands (`just --list`).
- You're tired of `.PHONY`, recursive `make` variable quoting, and silent partial failures.
- You need the same task runner to work across Linux, macOS, and CI with one binary.
- You want to call real build tools (cargo, npm, make, docker) from a clean top layer.

**Avoid when:**
- You need actual build semantics: incremental compilation, timestamp-based skipping, parallel target graphs — use `make`/`ninja`/language build tools.
- Your primary target is stock Windows with no `sh` and you don't want shebang recipes or a PowerShell shell override.
- Your project already standardizes on `package.json` scripts or a language-native task runner and adding a Rust binary is friction for the team.

## Alternatives

- go-task/task — YAML-defined, cross-platform task runner with built-in file-checksum "up to date" checks; use it when you want lightweight incremental behavior and prefer YAML to a `make`-like DSL.
- GNU make — use when you genuinely need a build system: file targets, mtime dependency resolution, and `-j` parallelism.
- sagiegurari/cargo-make — Rust-ecosystem task runner (`Makefile.toml`) with rich built-in tasks; use when your workflow is Cargo-centric and you want batteries included.
- casey/just is often paired with, not replaced by, `mise` (jdx/mise) for tool-version management plus tasks — use `mise` when you also need to pin toolchain versions.
- Plain shell scripts / npm scripts — use when the command set is tiny and a dedicated runner is overkill.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2016-06 | First public code; `make`-inspired command runner in Rust[^1]. |
| 1.0.0 | 2022-01 | Backwards-compatibility and stability commitment; no `just` 2.0[^3]. |
| 1.14.x | 2023 | `import` for textual inclusion of other justfiles. |
| 1.19.x | 2024 | `mod` modules introduced (initially behind `--unstable`). |
| 1.31.x | 2024 | Modules stabilized; multi-file justfile organization without unstable flag. |
| 1.55.0 | 2025 | `minimum-version` setting to gate on a required `just` version[^2]. |

## References

[^1]: casey/just repository and commit history. https://github.com/casey/just
[^2]: The `just` Programmer's Manual (reflects the latest release). https://just.systems/man/en/
[^3]: "Backwards Compatibility" — just README; 1.0 stability commitment. https://github.com/casey/just/blob/master/README.md#backwards-compatibility
[^4]: `rust-just` on npm (the `just` name is an unrelated JS project). https://www.npmjs.com/package/rust-just

## Tags

rust, command-runner, task-runner, justfile, make-alternative, cli, developer-tools, build-automation, devtools, cross-platform
