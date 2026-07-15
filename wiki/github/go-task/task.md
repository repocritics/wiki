# go-task/task

> A task runner and build tool that reads YAML `Taskfile.yml` and runs commands through an embedded Go shell interpreter, so bash-style recipes work identically on Linux, macOS, and Windows without external dependencies.

[GitHub repo](https://github.com/go-task/task) ·
[Official website](https://taskfile.dev) ·
[License: MIT](https://github.com/go-task/task/blob/main/LICENSE)

## Overview

Task is a Make alternative written in Go, first released in 2017 by Andrey Nering[^1]. It positions itself against GNU Make on two axes: it is a single static binary with no runtime dependency on a system shell, and its build files are YAML (`Taskfile.yml`) rather than Make's tab-sensitive syntax. As of 2026 it is one of the more widely adopted non-Make task runners, with roughly 15.8k GitHub stars and active weekly commits[^2].

The defining architectural choice — and the source of both its portability and its sharpest footguns — is that Task does not shell out to `/bin/sh` or `cmd.exe`. It embeds `mvdan.cc/sh`, a POSIX shell parser and interpreter written in Go[^3]. Recipes therefore run the same way on Windows as on Linux, with no WSL, MSYS2, or Git Bash required. The tradeoff is that the embedded interpreter is POSIX-ish, not full bash: arrays, process substitution, and some bashisms either fail or behave differently, and each command line in a task runs as a *separate* shell invocation.

Task's second identity is as a lightweight build system: it supports up-to-date checks (`sources`/`generates` fingerprinting) so tasks can be skipped when inputs are unchanged, which is what separates it from pure command runners like `just`. It is not a hermetic or sandboxed build system in the Bazel sense — there is no dependency graph enforcement, no remote caching, and no guarantee that declared `sources` are complete.

## Getting Started

```bash
# macOS / Linux (Homebrew)
brew install go-task/tap/go-task
# or: go install github.com/go-task/task/v3/cmd/task@latest
# or: npm install -g @go-task/cli
```

```yaml
# Taskfile.yml
version: '3'

vars:
  BINARY: myapp

tasks:
  build:
    desc: Compile the binary
    sources:
      - '**/*.go'
    generates:
      - '{{.BINARY}}'
    cmds:
      - go build -o {{.BINARY}} .

  test:
    deps: [build]      # deps run concurrently before this task
    cmds:
      - go test ./...

  default:
    cmds:
      - task: build
```

```bash
task            # runs the `default` task
task test       # runs build (dep) then test
task --list     # show tasks with `desc`
```

## Architecture / How It Works

A `Taskfile.yml` is parsed into a map of named tasks. Each task has `cmds` (shell lines), optional `deps` (other tasks run concurrently as prerequisites), and metadata such as `desc`, `dir`, `env`, and `vars`.

- **Embedded shell.** Every `cmds` line is executed by `mvdan.cc/sh`, not the host shell[^3]. Because each line is its own interpreter run, state does not persist between lines: `cd build` on one line and `make` on the next will *not* run `make` inside `build/`. The correct patterns are the task-level `dir:` key, a single `&&`-joined line, or a YAML block scalar (`|`).
- **Templating.** Values are run through Go's `text/template` engine, so `{{.VAR}}`, `{{.CLI_ARGS}}`, and sprig-style functions are available. Dynamic variables are computed by shelling out: `vars: { GIT_SHA: { sh: git rev-parse HEAD } }`.
- **Fingerprinting.** When a task declares `sources` and `generates`, Task computes a checksum (default) or timestamp of the sources and stores it under a `.task/` directory. On the next run, an unchanged fingerprint causes the task to be skipped. `method: none` disables it; `status:` lets you supply an arbitrary shell condition instead.
- **Composition.** `includes:` pulls in other Taskfiles under a namespace (`docs:build`). The experimental *Remote Taskfiles* feature extends this to `http(s)://` and git sources.
- **Experiments.** Breaking or speculative changes ship behind an opt-in experiments system gated by environment variables (e.g. remote Taskfiles, gentle `--force`, map variables), letting the v3 schema evolve without a v4 break[^4].

The dependency model is shallow by design: `deps` gives you concurrent prerequisites, and calling `task:` inside `cmds` gives you ordered sequencing. There is no global DAG scheduler that deduplicates or topologically orders the whole graph the way Make or Bazel does.

## Production Notes

- **The shell is not bash.** Recipes that rely on bash arrays, `<()` process substitution, `[[ ]]` extensions, or non-POSIX builtins can silently misbehave. The escape hatch is to invoke a real shell explicitly: `cmds: - bash -c '…'` (which reintroduces a bash dependency, defeating the portability win).
- **No state between command lines.** This is the most common first-week surprise. Set `dir:` at the task level, or join commands with `&&`, rather than expecting `cd`/`export` to carry over.
- **`.task/` cache hygiene.** The checksum directory must be git-ignored. Branch switches, partial builds, or edits that don't touch declared `sources` can leave a task incorrectly considered up-to-date; `task --force` (or `-f`) bypasses the check. Incomplete `sources` globs are the usual cause of "it didn't rebuild."
- **Variables are strings.** Task's variable model is fundamentally text; there are no first-class lists or maps outside the experimental map-variables feature. Complex data has to be encoded as strings or delegated to shell.
- **Remote Taskfiles execute remote code.** The remote-include experiment fetches and runs another author's task definitions; treat it like `curl | sh`. Task prompts on checksum changes, but pinning and review are on you.
- **Schema is versioned, tooling is not lockstep.** `version: '3'` files are not readable by v2 binaries and vice versa. CI images and contributor machines drifting across the v2→v3 boundary was a real migration cost for older projects[^4].
- **YAML footguns.** Templated strings frequently need quoting (`"{{.VAR}}"`), and deep indentation of multiline `cmds` is easy to get subtly wrong.

## When to Use / When Not

**Use when:**
- You want Make-like up-to-date skipping without Make's tab syntax, and cross-platform (including native Windows) recipes.
- Your team already ships Go tooling and wants a single static binary with zero install-time dependencies.
- Your build/dev automation is a modest set of commands with a few input/output relationships, not a large compilation graph.

**Avoid when:**
- You need a real build graph with hermetic inputs, remote caching, or correct incremental compilation of a large monorepo — reach for Bazel/Buck2 or a language-native build system.
- Your recipes are heavy bash scripts full of bashisms — the embedded interpreter will fight you.
- You only need to *run* named commands with no fingerprinting; a simpler command runner (`just`) is a closer fit and has no cache to reason about.

## Alternatives

- casey/just — command runner with a friendlier `justfile` syntax; use it when you want named recipes and no up-to-date/skip logic to reason about.
- GNU Make (mirrors/gnu-make) — the ubiquitous incumbent; use it when POSIX Make is already assumed everywhere and you want zero new binaries.
- magefile/mage — build tool where the logic is plain Go; use it when your team prefers writing build steps in Go over YAML.
- pyinvoke/invoke — Python task runner; use it when your project is Python-centric and tasks are Python functions.
- earthly/earthly — containerized, reproducible builds; use it when you need hermetic, cache-shareable CI builds rather than local convenience scripting.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2017 | Initial release; YAML task runner inspired by Make[^1]. |
| v2 | 2018 | Taskfile v2 schema; variables, dependencies, includes matured. |
| v3 | 2020 | Taskfile v3 schema — breaking changes to variables/includes; long-lived current major[^4]. |
| v3.x | 2021–2026 | Experiments framework: remote Taskfiles, gentle `--force`, map variables, ongoing under the v3 schema[^4]. |

## References

[^1]: go-task/task repository — created 2017-02-27. https://github.com/go-task/task
[^2]: GitHub API `repos/go-task/task` — ~15,837 stars, 867 forks, MIT, last push 2026-07-15 (fetched for this page). https://github.com/go-task/task
[^3]: `mvdan.cc/sh` — POSIX shell parser/interpreter in Go, the engine Task embeds to run commands. https://github.com/mvdan/sh
[^4]: Task documentation — Taskfile schema versions and the experiments system. https://taskfile.dev/experiments/

## Tags

go, build-tool, task-runner, make-alternative, devops, yaml, cross-platform, cli, automation, taskfile
