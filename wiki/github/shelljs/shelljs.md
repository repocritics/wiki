# shelljs/shelljs

> Portable Unix shell commands (`cp`, `rm`, `sed`, `grep`, `exec`…) reimplemented as synchronous Node.js function calls.

[GitHub repo](https://github.com/shelljs/shelljs) ·
[npm package](https://www.npmjs.com/package/shelljs) ·
[License: BSD-3-Clause](https://github.com/shelljs/shelljs/blob/main/LICENSE)

## Overview

ShellJS lets you write build scripts and CLI tooling in JavaScript while keeping the vocabulary of a Unix shell. Instead of shelling out to `bash`, you call `shell.cp('-R', src, dst)`, `shell.sed('-i', /foo/, 'bar', file)`, `shell.rm('-rf', dir)` — each a synchronous function that mimics the corresponding POSIX command, including common flags. The pitch is portability: the same script runs on Windows, Linux, and macOS without a `sh` interpreter installed[^1].

It first appeared in 2012, authored by Artur Adib while at Mozilla, and became the standard way to write cross-platform npm build scripts in the pre-`npm run` / pre-`cross-env` era. It was embedded in JSHint, ESLint, Yeoman, and countless `Makefile` replacements[^1]. As of 2026 it has ~14.4k stars and remains widely depended-upon, but it occupies a mature, slow-moving niche: the async-first, template-literal generation of shell tools (zx, execa, Bun Shell) has moved past its synchronous design.

The defining tradeoff is exactly that synchronicity. Every command blocks the event loop until it completes, which makes scripts read top-to-bottom like a shell session — and makes ShellJS unsuitable for anything concurrent or server-side. It is a scripting library, not an application library.

## Getting Started

```bash
npm install shelljs        # library
npm install -g shelljs     # or global, to script outside a project
```

```javascript
const shell = require('shelljs');

if (!shell.which('git')) {
  shell.echo('Sorry, this script requires git');
  shell.exit(1);
}

shell.rm('-rf', 'out/Release');
shell.cp('-R', 'stuff/', 'out/Release');

shell.cd('lib');
shell.ls('*.js').forEach((file) => {
  shell.sed('-i', 'BUILD_VERSION', 'v0.1.2', file);
});
shell.cd('..');

// Run an external tool synchronously, inspect the exit code
if (shell.exec('git commit -am "Auto-commit"').code !== 0) {
  shell.echo('Error: Git commit failed');
  shell.exit(1);
}
```

For command-line use without writing JS, the sibling project `shelljs/shx` exposes the same commands directly to the shell (`shx rm -rf foo`)[^2].

## Architecture / How It Works

Every command returns a **ShellString** — a `String` subclass carrying `.stdout`, `.stderr`, and `.code`, plus chainable helpers like `.to(file)`, `.toEnd(file)`, `.grep()`, and `.cat()`. This is what lets `cat('f').grep('x').to('out')` read like a shell pipeline while still being ordinary method calls. Commands that "succeed or fail" (e.g. `cp`, `rm`, `mkdir`) return a ShellString whose `.code` and `.stderr` signal the result; by default they do **not** throw[^3].

Global behavior is controlled through `shell.config`: `fatal` (throw/exit on error), `verbose` (echo commands), `silent` (suppress output), and glob options. `set('-e')` / `set('-f')` toggle these the way `set` does in a real shell. Globbing itself is delegated to `fast-glob`, so `*`, `?`, and `**` patterns work across platforms[^1].

The whole library is deliberately synchronous. File operations use Node's `fs` sync APIs and read whole files into memory; process execution uses blocking spawns. `exec()` runs a command string through the system shell (`sh` on Unix, `cmd.exe` on Windows) — which means it inherits shell parsing, and with it command-injection risk on unsanitized input. The newer `cmd()` command is the safer counterpart: it takes an argument array, runs exactly one executable via a `spawnSync`-style call, and treats `|`, `;`, `&` as literal characters rather than shell operators[^3]. `cmd()` does globbing itself (disable with `set('-f')`) and has no async mode.

A plugin API (`plugin.register`) lets third parties add commands that behave like built-ins, sharing the ShellString return convention and config object[^4].

## Production Notes

- **This blocks the event loop, always.** ShellJS is for build scripts, generators, and CLIs — never inside a request handler, worker, or anything that needs concurrency. There is no async mode except `exec({async:true})`.
- **`exec()` is a command-injection surface.** It passes your string to the system shell verbatim. Interpolating user input, branch names, filenames, or env values into an `exec()` string is a real vulnerability. Prefer `cmd('git', 'checkout', branch)` — the argument-array form cannot be shell-injected. The maintainers publish explicit security guidance on this[^5].
- **`exec()` semantics differ from bash.** It runs through `sh` (or `cmd.exe`), not `bash`, so bashisms silently misbehave. Pass `{shell: '/bin/bash'}` if you need them.
- **Performance degrades on large loops.** Each command re-resolves globs and, for `exec`, spawns a fresh shell process. A script that `exec`s per-file over thousands of files pays a process-spawn tax on every iteration; batch where possible, or drop to `cmd()` / native `fs`.
- **Whole-file, in-memory processing.** `cat`, `sed`, `sort`, `grep` load entire files into memory — fine for source files, wrong for multi-gigabyte data. `sed` runs line-by-line and cannot match patterns that span newlines (documented behavior, not a bug)[^3].
- **`sed('-i', …)` makes no backup.** Unlike some `sed` implementations, in-place edits are destructive with no `.bak`. Combine with version control or copy first.
- **Windows caveats.** `chmod` maps imperfectly onto the Windows permission model; under WSL it follows POSIX correctly. `exec` uses `cmd.exe`, so scripts leaning on `sh` builtins break there even though the ShellJS commands themselves are portable.
- **Error handling is opt-in.** Commands don't throw by default — you must check `.code`, or set `config.fatal = true` (`set('-e')`) to make failures exit. Silent-failure bugs are common in scripts that never inspect the return value.
- **Global import is discouraged.** `require('shelljs/global')` still works but pollutes the global namespace (`cd`, `rm`, `ls` as bare globals); the maintainers now recommend the local `require('shelljs')` / `import shell from 'shelljs'` form[^1].

## When to Use / When Not

**Use when:**
- You're writing cross-platform build/release scripts and want one file that runs identically on Windows and Unix without a shell dependency.
- You prefer readable, top-to-bottom synchronous scripts over async orchestration.
- You want familiar Unix flags (`-rf`, `-R`, `-i`) without reimplementing them on `fs`.

**Avoid when:**
- You need concurrency, streaming, or anything running in a server/process that can't block.
- Your workload is process-heavy and you want modern async execution with good error/stream handling — use execa or zx.
- You're mostly running external commands (not doing `fs`-style ops) — a dedicated process library is a better fit than the shell-command veneer.

## Alternatives

- `google/zx` — use instead when you want modern async shell scripting with template-literal command syntax and ESM; the de facto successor for new scripts.
- `sindresorhus/execa` — use instead when your job is running external processes robustly (streams, promises, cross-platform argument handling) rather than emulating `cp`/`rm`/`sed`.
- `shelljs/shx` — use instead when you only need ShellJS commands inline in `package.json` scripts, not in a `.js` file (same project, CLI wrapper).
- `jprichardson/node-fs-extra` — use instead when you only need enhanced filesystem operations (`copy`, `remove`, `ensureDir`) and no shell/exec layer.
- `nodejs/node` `node:child_process` + `node:fs/promises` — use instead when you'd rather depend on the standard library and can accept writing the portability shims yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-03 | First commit; Unix-command emulation for Node[^1]. |
| 0.7.0 | 2016-04-25 | Pre-`cmd()` era; the long-lived 0.7 line[^6]. |
| 0.8.0 | 2018-01-12 | Plugin API, config refinements[^6]. |
| 0.8.4 | 2020-04-25 | Late 0.8 maintenance release[^6]. |
| 0.8.5 | 2022-01-14 | Final 0.8 release; then a multi-year quiet period[^6]. |
| 0.9.0 | 2025-03-08 | Project revival: `cmd()` safe-exec command, Node baseline raised[^6]. |
| 0.10.0 | 2025-05-09 | Current line; tested on every LTS Node since v18[^6]. |

## References

[^1]: ShellJS README — features, portability, global-vs-local import guidance. https://github.com/shelljs/shelljs
[^2]: shx — ShellJS commands exposed to the command line. https://github.com/shelljs/shx
[^3]: ShellJS command reference — ShellString return type, `cmd()` vs `exec()`, `sed` line-by-line behavior. https://github.com/shelljs/shelljs
[^4]: ShellJS plugin API (wiki). https://github.com/shelljs/shelljs/wiki/Using-ShellJS-Plugins
[^5]: ShellJS security guidelines — command injection and `exec()`. https://github.com/shelljs/shelljs/wiki/Security-guidelines
[^6]: ShellJS GitHub releases (dates from published release tags). https://github.com/shelljs/shelljs/releases

## Tags

javascript, nodejs, shell, unix, cli, build-tools, cross-platform, scripting, filesystem, process-execution
