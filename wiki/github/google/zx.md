# google/zx

> A JavaScript wrapper around child_process that makes shell scripting in Node feel like Bash — with argument escaping and cross-platform defaults.

[GitHub repo](https://github.com/google/zx) ·
[Official website](https://google.github.io/zx/) ·
[License: Apache-2.0](https://github.com/google/zx/blob/main/LICENSE)

## Overview

zx is a small library, plus a CLI, that lets you write shell scripts in JavaScript instead of Bash. Its central idea is the `$` tagged template: `` await $`git status` `` spawns a command, waits for it, and returns its output as an object. It was written by Anton Medvedev and open-sourced under the Google org in 2021[^1]; the README carries the standard "not an officially supported Google product" disclaimer[^2], so treat it as a well-maintained community project that happens to live under `google/`, not something with a Google support contract.

The pitch is narrow and honest: Bash is good at gluing processes together but awkward for control flow, arrays, JSON, and error handling; Node is the reverse, but `child_process` is verbose and its quoting rules are a footgun. zx sits in the middle. You get JavaScript's data structures, `async`/`await`, and `Promise.all` for concurrency, while `$` handles spawning, streaming, and — importantly — automatically quotes interpolated values so `` $`rm ${file}` `` cannot be turned into command injection by a space or semicolon in `file`.

The defining tension is that zx is a *scripting DSL*, not a general process library. It runs commands through a real shell by default (bash, or PowerShell on Windows), it installs a set of global helpers, and it optimizes for terse throwaway automation over long-lived application code. That makes it excellent for build glue, CI steps, and one-off tooling, and a poor fit for library internals where you want an explicit, dependency-light process call.

## Getting Started

```bash
npm install zx
```

```js
#!/usr/bin/env zx

// Run the script with: zx ./deploy.mjs
const branch = await $`git branch --show-current`
const name = 'foo bar'          // interpolation is auto-quoted — no injection
await $`mkdir /tmp/${name}`      // runs: mkdir '/tmp/foo bar'

// Concurrency is just Promise.all
await Promise.all([
  $`sleep 1; echo 1`,
  $`sleep 2; echo 2`,
])

// Handle failure without try/catch
const p = await $`exit 3`.nothrow()
if (p.exitCode !== 0) console.log('command failed:', p.exitCode)
```

Used as a library instead of via the CLI, import the pieces explicitly so you do not rely on injected globals:

```js
import { $, cd, fs, glob } from 'zx'
```

## Architecture / How It Works

`` $`...` `` is a tagged template that returns a **ProcessPromise** — a thenable that eventually resolves to a **ProcessOutput** (`stdout`, `stderr`, `exitCode`, plus `toString()`). Because it is a Promise, `await` works; because it is also a builder, you can chain `.pipe()`, `.nothrow()`, `.quiet()`, `.timeout()`, and `.stdin` before awaiting. Under the hood each call is a `child_process.spawn` of a shell process.

Argument safety comes from per-substitution quoting. Every `${value}` in the template is passed through `$.quote` (a shell-quoting function) before the command string is assembled, so interpolated data is treated as a literal argument, not as shell syntax. Arrays are expanded element-by-element. The escape hatch is that you can still build a command as a plain string and hand it to `$`, which bypasses the protection — the safety is a property of interpolation, not of the whole API.

Configuration lives on the `$` object itself: `$.shell` (which shell binary to invoke), `$.prefix` (prepended to every command — defaults to `set -euo pipefail;` so failures and unset variables abort the pipeline), `$.verbose`, `$.env`, and `$.cwd`. `within()` runs a callback with a temporarily overridden configuration, and `cd()` changes the working directory for subsequent commands.

The package bundles a set of conveniences so scripts do not need their own dependencies: `fs` (fs-extra-style), `glob`, `chalk`-style coloring, `which`, `question`, `sleep`, `retry`, `spinner`, `fetch`, and a Markdown mode where the CLI extracts and runs ` ```js ` code blocks from a `.md` file. The CLI also accepts remote scripts via URL and `--eval` snippets. Over successive major versions the project moved to TypeScript and ESM and progressively inlined or dropped third-party runtime dependencies, so recent versions install as an effectively self-contained bundle[^3].

## Production Notes

**It needs a shell, and the default shell is bash.** On a minimal container (Alpine ships `sh`, not `bash`) the default `$.prefix` of `set -euo pipefail;` and bash invocation will fail. Either install bash, or set `$.shell = '/bin/sh'` and clear/adjust `$.prefix` — `pipefail` is not POSIX `sh`. This is the single most common surprise when moving a working local script into CI or a slim image.

**Escaping protects interpolation, not string-building.** `` $`echo ${userInput}` `` is safe; `$('echo ' + userInput)` or hand-assembling a command and passing it in is not. Reviewers should look for places where a command is concatenated as a string rather than interpolated through the template.

**Globals are a scripting convenience, not an application pattern.** Running via the `zx` CLI injects `$`, `cd`, `fs`, etc. into global scope. That is fine for scripts but you should not depend on it in imported modules; use explicit `import { $ } from 'zx'` there. Mixing the two leads to confusing "works as a script, breaks as a module" bugs.

**Performance.** Each `$` call spawns a shell process. That is negligible for a deploy script but real if you call it thousands of times in a loop — batch work into a single command or drop to a lower-level exec when it matters.

**Cross-runtime claims come with caveats.** zx advertises Node, Bun, Deno, and GraalVM support, but shell availability, path handling, and streaming behavior differ across runtimes and especially on Windows (PowerShell quoting is not bash quoting). Test on the runtime and OS you actually deploy to rather than trusting the compatibility matrix.

**Upgrade friction.** The v6 → v7 → v8 line changed the import surface, tightened or removed deprecated globals, and changed how bundled helpers like `fetch` are provided[^3]. Pin the major version in CI and read the release notes before bumping; a script that relied on an old global or an old `fetch` shape can break silently on a major upgrade.

## When to Use / When Not

**Use when:**
- You are writing build, release, or CI glue that shells out to several tools and want JavaScript control flow and data handling.
- You want automatic argument escaping instead of hand-quoting Bash.
- You want concurrency (`Promise.all`) and readable error handling over Bash's subshell and `set -e` idioms.
- The script is a script — short-lived, run by developers or CI, not shipped as a library.

**Avoid when:**
- You are writing library or application code that spawns processes; reach for a lower-level, no-shell exec call instead.
- You target minimal environments where installing bash is undesirable.
- You need a locked-down, no-shell execution model for security reasons — running through a shell is inherent to zx's design.
- You want a long-term-stable API with rare breaking changes; zx iterates and has had non-trivial major migrations.

## Alternatives

- sindresorhus/execa — lower-level process execution with no shell DSL and no injected globals; use it when you want a library primitive, not a scripting language.
- shelljs/shelljs — reimplements Unix commands in pure JS so no external shell is required; use it when portability without bash matters more than template ergonomics.
- dsherret/dax — a zx-style cross-runtime shell built Deno-first (also works on Node); use it when Deno is your primary target.
- oven-sh/bun — Bun ships a built-in `$` shell; use it when you are already all-in on the Bun runtime and want zero extra dependencies.
- Plain Bash + make — use it when the task is simple, the environment already has a shell, and adding a Node dependency is not worth it.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-05 | Open-sourced under the google org; single-file `$` wrapper[^1]. |
| 6.x | 2022 | Move to TypeScript and ESM-first packaging[^3]. |
| 7.x | 2022 | Reworked bundled dependencies and helper surface[^3]. |
| 8.x | 2024 | Self-contained bundle, smaller install, `zx@lite` variant introduced[^3]. |

(Exact release dates vary by patch; see the GitHub releases page for the authoritative changelog.)

## References

[^1]: google/zx repository, created 2021-05-05. https://github.com/google/zx
[^2]: zx README — "This is not an officially supported Google product." https://github.com/google/zx#license
[^3]: zx releases and changelog. https://github.com/google/zx/releases

## Tags

javascript, nodejs, shell, scripting, cli, child-process, automation, bash, devtools, apache-2.0
