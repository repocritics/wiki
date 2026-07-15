# sindresorhus/execa

> Process execution for Node.js — a `child_process` wrapper that trades minimalism for a large, ergonomic surface.

[GitHub repo](https://github.com/sindresorhus/execa) ·
[License: MIT](https://github.com/sindresorhus/execa/blob/main/license)

## Overview

Execa is a Node.js library for running external commands, built directly on top of the built-in [`child_process`](https://nodejs.org/api/child_process.html) module[^1]. It exists because `child_process` is low-level and full of sharp edges: manual buffering of stdout/stderr, no promise interface, inconsistent Windows behavior, error objects that omit the command that failed, and easy shell-injection footguns. Execa papers over all of these with a promise-returning API, structured error objects, cross-platform normalization, and (since v9) a tagged-template syntax that reads like a shell script without invoking a shell[^2].

It is one of the most-depended-upon packages in the JavaScript ecosystem — pulled in transitively by a large share of CLI tooling, test runners, and build scripts. Maintained by Sindre Sorhus (author of hundreds of npm packages) with @ehmicky as the primary contributor behind the v7–v9 rewrites, its stability and ubiquity are its main selling points; you are rarely the first to hit a bug.

The defining tension is scope. Execa is not a thin shim — the v9 line added inter-process messaging, web/Node stream interop, output transforms, line iteration, graceful cancellation, and a `$` script mode. That breadth is convenient but means the package is heavier and its API larger than callers who "just want to run a command" expect. It is also strictly ESM-only since v6, which remains the single most common upgrade blocker.

## Getting Started

```sh
npm install execa
```

```js
import {execa} from 'execa';

// Template syntax: arguments are interpolated safely, no shell involved.
const {stdout} = await execa`git rev-parse --short HEAD`;
console.log(stdout);

// Errors are thrown with full context.
try {
	await execa`git push origin ${branch}`;
} catch (error) {
	console.error(error.shortMessage); // includes command, exit code, stderr
}
```

```js
// Script mode ($): closer to shell scripting, still no injection risk.
import {$} from 'execa';

const name = 'foo bar';               // spaces are handled, not word-split
await $`mkdir /tmp/${name}`;
const {stdout} = await $`cat package.json`.pipe`grep version`;
```

## Architecture / How It Works

Execa is a wrapper, not a reimplementation: every command still bottoms out in `child_process.spawn`. The value is in the layers around it.

- **Argument handling.** By default execa does *not* spawn a shell. Arguments are passed as an array to `spawn`, so interpolated values cannot be reinterpreted as shell syntax — the reason "no escaping/quoting needed, no injection risk" holds. Opting into `shell: true` re-enables shell parsing and forfeits that guarantee. On Windows, execa uses `cross-spawn` to normalize `PATH`/`PATHEXT` resolution and shebang handling so that the same call works as it does on POSIX systems.
- **Output collection.** Execa buffers stdout/stderr into the resolved result by default, applying a `maxBuffer` cap (historically 1000 MB / 100 MB depending on version) and optional line-splitting, stripping, and encoding. Output can instead be streamed, redirected to files, iterated line-by-line via async iteration, or run through generator-based transform functions.
- **Result and error model.** Success resolves to a result object (`stdout`, `stderr`, `exitCode`, `durationMs`, `command`, …). Failure rejects with an `ExecaError` carrying the same fields plus `shortMessage`, `failed`, `timedOut`, `isCanceled`, `isTerminated`, and the underlying cause. This structured error is a large part of why execa is preferred over raw `child_process`.
- **Piping.** `.pipe` chains subprocesses in-process (not via a shell pipe), and the result exposes `pipedFrom` so intermediate outputs remain inspectable — something shells discard.
- **v9 additions.** Typed IPC (`sendMessage`/`getOneMessage`/`ipcInput`/`ipcOutput`), conversion to/from web and Node streams, `.duplex()`, and `gracefulCancel` via an `AbortSignal`.

The API is deliberately option-heavy: most behavior is controlled by a single options object that can be bound once (`execa({...})`) and reused as a template tag.

## Production Notes

- **ESM-only since v6.** Execa 6+ ships no CommonJS build; `require('execa')` throws. Projects still on CJS must stay on execa 5 (which is stable but frozen) or migrate to ESM. This is the dominant real-world upgrade pain and the reason many codebases lag several majors behind.
- **`maxBuffer` truncation.** Commands that emit large output can hit the buffer cap and reject with `isMaxBuffer: true`, or silently truncate if you're reading the result field without checking. For unbounded output, stream instead of buffering.
- **Not for hot loops.** Every call spawns an OS process. Execa's overhead over raw `child_process` is negligible relative to `spawn` itself, but spawning thousands of short-lived processes is slow regardless of library — batch work inside one process where possible.
- **`shell: true` reintroduces injection.** The safety guarantee is specifically about the default (no-shell) path. Any code that flips on `shell` must escape untrusted input itself.
- **Windows still leaks.** `cross-spawn` closes most gaps, but signal semantics differ (Windows has no real `SIGTERM`), `.kill()` behavior is not identical to POSIX, and graceful termination relies on execa's own logic rather than the OS.
- **Version cadence is aggressive.** Majors ship yearly with real breaking changes (ESM in 6, Node-version floor bumps in 8, API reshaping in 9). Pin and read the migration notes; do not float across majors.
- **Bundle weight.** Execa and its transitive deps are non-trivial. For size-sensitive contexts (serverless cold starts, published libraries), the maintainer's own `nano-spawn` is the intended lighter alternative.

## When to Use / When Not

**Use when:**
- You run external commands from Node scripts, CLIs, or tests and want promises, structured errors, and cross-platform behavior for free.
- You want shell-like ergonomics (template/`$` syntax, piping) without a shell and its injection risk.
- You need stream interop, line iteration, output transforms, or process IPC.

**Avoid when:**
- Your project is CommonJS and cannot move to ESM (stay on execa 5 or use `nano-spawn`/raw `child_process`).
- You want the smallest possible dependency footprint.
- You only ever run one trivial command — `node:util`'s `promisify(exec)` may be enough.
- You need synchronous, filesystem-heavy shell scripting — ShellJS fits that shape better.

## Alternatives

- google/zx — shell scripting with a similar `$` template syntax; leans toward interactive scripts, runs a shell by default, less strict about injection safety.
- sindresorhus/nano-spawn — same author; a minimal spawn wrapper for when execa's surface and size are more than you need.
- moxystudio/node-cross-spawn — the low-level Windows-normalizing spawn that execa itself uses; choose it when you only need cross-platform `spawn` and nothing else.
- shelljs/shelljs — portable synchronous Unix shell commands reimplemented in JS; use when you want `cp`/`grep`/`sed`-style helpers rather than arbitrary process execution.
- node:child_process / node:util promisify(exec) — the built-in; use when adding a dependency isn't worth it and you accept the manual buffering, error, and Windows handling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-04 | Promise-based API stabilized on `child_process`. |
| 5.0 | 2021-04 | Last version shipping a CommonJS build. |
| 6.0 | 2021-11 | ESM-only; `require` no longer supported. |
| 7.0 | 2022-10 | API and internals reworked ahead of the v9 feature set. |
| 8.0 | 2023-08 | Dropped older Node.js versions; further internal cleanup. |
| 9.0 | 2024-05 | Major expansion: template/`$` syntax, typed IPC, stream interop, transforms, graceful cancel[^2]. |

## References

[^1]: Node.js documentation, "Child process". https://nodejs.org/api/child_process.html
[^2]: Execa README and API reference, `sindresorhus/execa`. https://github.com/sindresorhus/execa

## Tags

javascript, nodejs, child-process, process-execution, cli, shell, spawn, streams, esm, developer-tools
