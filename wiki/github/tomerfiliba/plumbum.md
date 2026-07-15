# tomerfiliba/plumbum

> Shell combinators for Python — run local and remote commands as Python objects, with pipes, redirection, paths, and a CLI toolkit.

[GitHub repo](https://github.com/tomerfiliba/plumbum) ·
[Official website](https://plumbum.readthedocs.io) ·
[License: MIT](https://github.com/tomerfiliba/plumbum/blob/master/LICENSE)

## Overview

Plumbum (Latin for *lead*, the metal old pipes were made from) is a Python
library by Tomer Filiba, first released in 2012[^1], whose slogan is "Never
write shell scripts again." It gives you the compactness of shell — pipes,
redirection, backgrounding, path juggling — as ordinary Python expressions,
so you can replace fragile Bash glue with code that has real control flow,
error handling, and cross-platform behavior[^2].

Its central idea is that an external program is a first-class Python object.
`local["ls"]` (or `from plumbum.cmd import ls`) is a *bound command*; indexing
it with arguments (`ls["-la"]`) returns a new command; calling it (`ls()`)
runs it and returns captured stdout as a string. Operators are overloaded to
mirror the shell: `|` pipes, `<`/`>`/`>>` redirect, `&` with the `FG`/`BG`
tokens runs in foreground or background. The same command object works locally
or over SSH via `SshMachine`, which is the feature that most distinguishes it
from `subprocess`-wrapping alternatives.

The defining tension is *shell mimicry vs. Pythonic clarity*. The operator
overloading reads well in a REPL but hides machinery: exit codes raise
`ProcessExecutionError` by default, output is captured (not streamed) unless
you ask otherwise, and the DSL diverges from `subprocess` semantics enough
that debugging quoting or environment means learning Plumbum's model rather
than the shell's. It is a mature, single-maintainer project — steady rather
than fast-moving — whose scope is deliberately broad: process running, path
abstraction, a CLI-argument framework, and terminal colors all ship together.

## Getting Started

```bash
pip install plumbum
# optional: pip install "plumbum[ssh]"  # paramiko backend for remote machines
```

```python
from plumbum import local, FG
from plumbum.cmd import ls, grep, wc

# run a command, capture stdout as a str
print(ls("-l"))

# pipe: ls -a | grep -v '\.py' | wc -l
chain = ls["-a"] | grep["-v", r"\.py"] | wc["-l"]
print(chain())                 # -> '27\n'

# stream to the real stdout instead of capturing
(ls["-a"] | grep[r"\.py"]) & FG

# scoped working directory and environment
with local.cwd("/tmp"), local.env(DEBUG="1"):
    print(local.cmd.pwd())
```

```python
# remote execution over SSH — same command API
from plumbum import SshMachine
with SshMachine("host", user="john", keyfile="/path/to/id_rsa") as rem:
    r_grep = rem["grep"]
    print(rem["ls"]("/var/log"))
```

## Architecture / How It Works

A *machine* (`local`, `SshMachine`, `ParamikoMachine`) is the root object.
It resolves executables against `PATH`, owns the working directory and
environment, and produces command objects bound to that machine. `plumbum.cmd`
is an import hook: `from plumbum.cmd import git` is sugar for `local["git"]`.

Commands are immutable and composable. `cmd["arg"]` returns a `BoundCommand`;
`a | b` builds a `Pipeline`; `cmd < file` / `cmd > file` build redirection
wrappers. Nothing runs until you *invoke* — `cmd()` (capture), `cmd & FG`
(inherit stdio), `cmd & BG` (returns a `Future`), `cmd.run(retcode=None)`
(get the `(exit, stdout, stderr)` tuple), or `cmd.popen()` (raw `Popen`).
By default a non-zero exit raises `ProcessExecutionError`; you opt out per
call with `retcode=None` or a set of allowed codes.

Paths are their own abstraction. `local.path("/etc")` yields a `LocalPath`
supporting `/` joining, globbing, and stat helpers, and `SshMachine` exposes
matching `RemotePath` objects so the same traversal code works over the wire.
This overlaps `pathlib`; the two are similar but not identical.

Two subsystems are effectively independent libraries in the same package:
`plumbum.cli` is a class-based CLI framework (`cli.Application`, `cli.Flag`,
`cli.SwitchAttr`, the `@cli.switch` decorator, subcommands) that competes with
argparse/click, and `plumbum.colors` is an ANSI color/style DSL. Either can be
used without touching process execution.

## Production Notes

- **Output is captured by default.** `cmd()` buffers all stdout in memory and
  returns it as a `str`. For long-running or high-volume commands this can
  balloon memory and gives no live progress — use `& FG` to inherit stdio,
  `.run(...)` to also get stderr, or `.popen()` to stream manually.
- **Exit codes raise.** A non-zero return throws `ProcessExecutionError`.
  Code ported from shell (where `grep` "failing" to match is routine) needs
  explicit `retcode=None` or `retcode=(0, 1)`, or it will surface as an
  exception in unexpected places.
- **Quoting is Plumbum's, not the shell's.** Arguments are passed as an argv
  list, not through `/bin/sh`, so shell features (glob expansion, `$VAR`,
  `~`, `&&`, subshells) do **not** apply inside a command. This is safer
  (no injection) but trips people expecting `cmd["*.txt"]` to expand.
- **SSH cost model.** Each `SshMachine` command spawns a new SSH process;
  there is no persistent shell, so tight loops of remote commands pay
  per-invocation latency. `ParamikoMachine` avoids re-forking `ssh` but pulls
  in the `paramiko` dependency and its own quirks.
- **Windows is supported but uneven.** Paths, `local`, and CLI work cross-
  platform, but many scripts assume POSIX tools; remote execution and some
  redirection idioms behave differently there.
- **v2.0 is a recent boundary.** The 2.0 line (2026) is the current major
  series; pin your version and read the changelog before upgrading long-lived
  automation, as 1.x → 2.x is a deliberate break point[^3].

## When to Use / When Not

**Use when:**
- You are replacing a Bash script with real logic, error handling, or tests.
- You need the *same* code to run commands locally and over SSH.
- You want a compact, readable process DSL in an interactive/REPL setting.
- You want a lightweight CLI framework (`plumbum.cli`) without argparse boilerplate.

**Avoid when:**
- You need to stream large output or interact live with a subprocess — raw
  `subprocess` or `pexpect` gives finer control.
- You want to minimize dependencies for a single, simple command call.
- Your team already standardizes on `subprocess.run`/`pathlib` and the DSL
  would be an unfamiliar layer for maintainers.
- You need heavy parallel/async orchestration — Plumbum's model is synchronous.

## Alternatives

- amoffat/sh — very similar "commands as functions" philosophy, POSIX-focused, no built-in SSH/CLI framework; use when you want a lighter, more magical local runner.
- python/cpython `subprocess` — the standard library; use when you want zero deps and full control over streams and quoting.
- pallets/click — dedicated CLI framework; use instead of `plumbum.cli` when the argument parser is the whole point.
- fabric/fabric — SSH automation and task running built on paramiko/invoke; use when remote orchestration across many hosts is the core need.
- pyinvoke/invoke — task execution and shell command running; use when you want a make-like task runner rather than a command DSL.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-04-27 | First public commit; "shell combinators" concept[^1]. |
| 1.6.x | 2015–2020 | Long-lived stable series across the Python 2/3 era. |
| 1.7.0 | 2021-02-08 | Packaging modernization; dropped legacy Python support[^3]. |
| 1.8.0 | 2022-10-06 | Continued 1.x maintenance line. |
| 1.9.0 | 2024-10-05 | Feature/maintenance release. |
| 1.10.0 | 2025-10-31 | Final 1.x-series release. |
| 2.0.0 | 2026-06-04 | Current major series; intentional break from 1.x[^3]. |

## References

[^1]: Plumbum repository, created 2012-04-27. https://github.com/tomerfiliba/plumbum
[^2]: Plumbum documentation, "Plumbum: Shell Combinators." https://plumbum.readthedocs.io/en/latest/
[^3]: Plumbum releases (dates and tags from the GitHub Releases API). https://github.com/tomerfiliba/plumbum/releases

## Tags

python, shell, subprocess, ssh, cli, process-execution, automation, devops, command-line, scripting, cross-platform
