# pyinvoke/invoke

> A Python task runner and subprocess wrapper — the task-execution core extracted from Fabric, usable on its own.

[GitHub repo](https://github.com/pyinvoke/invoke) ·
[Official website](https://pyinvoke.org) ·
[License: BSD-2-Clause](https://github.com/pyinvoke/invoke/blob/main/LICENSE)

## Overview

Invoke is a Python library that does two things at once: it runs shell
subprocesses with sane defaults (`c.run("...")`), and it organizes plain Python
functions into a CLI you invoke from the command line (`invoke build test`). It
grew out of Fabric 1.x, whose task-running and command-execution machinery was
split off so it could be reused without Fabric's SSH/deployment concerns.
Modern Fabric (2.x+) is built on top of Invoke[^1].

The audience is Python developers who want a `Makefile` replacement written in
Python — the same role as `make`, `rake`, or `just`, but with tasks as real
Python functions that receive a `Context` object and can call each other. Unlike
`argparse`/`click`, Invoke ships its own argument parser and its own hierarchical
configuration system (YAML/JSON/env/CLI layered together), which is the source of
both its convenience and most of its idiosyncrasies.

The defining tension: Invoke is stable and dependency-free, but it moves slowly.
It is largely maintained by one person (Jeff Forcier / bitprophet)[^2], carries
a large open-issue backlog (400+), and made deliberate design choices — its own
parser, no native task parallelism — that diverge from what users of `click` or
`argparse` expect. It is dependable rather than fast-moving.

## Getting Started

```bash
pip install invoke
```

```python
# tasks.py
from invoke import task

@task
def clean(c):
    c.run("rm -rf build dist")

@task(pre=[clean])
def build(c, docs=False):
    c.run("python -m build")
    if docs:
        c.run("sphinx-build docs docs/_build")

@task
def test(c, verbose=False):
    flag = "-v" if verbose else ""
    c.run(f"pytest {flag}", pty=True)
```

```bash
invoke --list              # show available tasks
invoke build --docs        # runs clean (pre-task) then build
invoke test --verbose      # boolean flag -> --verbose
```

## Architecture / How It Works

The core objects are `Task`, `Collection`, `Context`, and `Config`.

- **`@task`** wraps a function into a `Task`. The first positional parameter is
  always a `Context`; remaining parameters become CLI flags. Type inference is
  minimal — a parameter defaulting to `False` becomes a boolean `--flag`,
  everything else takes a string value.
- **`Collection`** is a namespace of tasks. Modules are auto-collected into a
  root collection; sub-collections produce dotted task names
  (`docs.build`). This is how Invoke supports namespaced task trees without a
  registry.
- **`Context`** is the per-run object handed to every task. `c.run()` executes a
  subprocess; `c.cd()` and `c.prefix()` are context managers that mutate the
  environment for nested `run` calls. Context also exposes merged config.
- **`Config`** merges settings from multiple layers in a fixed precedence order:
  internal defaults, collection-level config, user config file, project config
  file, environment variables (`INVOKE_*`), and CLI overrides. This layering is
  what Fabric extends to add SSH host config.

Command execution is built on a `Runner` abstraction. The default `Local`
runner uses `subprocess`, optionally allocating a pseudo-terminal (`pty=True`)
so interactive/color-aware programs behave correctly. Runners handle stream
mirroring (child stdout/stderr echoed live while also captured), which is why
`c.run(...).stdout` works even for long-running commands.

Argument parsing is **not** `argparse` or `click` — Invoke ships its own parser
that supports per-task flags, global flags before the task name, and flag
inheritance. This is deliberate (it enables the `invoke task1 task2` multi-task
call syntax) but means Invoke does not benefit from the broader `click`
ecosystem.

## Production Notes

- **Task parallelism is not built in.** Tasks run sequentially in the order
  given. Parallel task execution has been a long-standing roadmap item rather
  than a shipped feature; if you need it, you parallelize *inside* a task
  (threads, `concurrent.futures`, or spawning background subprocesses)[^3].
- **Passing arguments between tasks is awkward.** `pre`/`post` hooks run tasks
  with their defaults; to call a task with specific arguments from another task
  you invoke the underlying function directly (`build(c, docs=True)`) rather
  than through the task machinery. This trips up newcomers expecting a
  make-style dependency graph.
- **The parser has sharp edges.** Because Invoke rolls its own, quirks appear
  around flags that take values, `--`-terminated argument passthrough, and
  boolean vs. valued inference. Read the parsing docs before assuming
  `click`-like behavior.
- **`pty=True` changes capture semantics.** Under a pty, stdout and stderr are
  merged into one stream and you lose separate `.stderr`. Windows lacks real
  pty support, so `pty=True` is effectively a no-op there — a portability
  footgun for cross-platform task files.
- **Release cadence is slow.** Fixes and features can sit for long periods
  given the solo-maintainer model. Pin versions; do not assume a bug reported
  upstream will be resolved on a near-term timeline[^2].
- **2.0 dropped Python 2.** The 1.x line supported Python 2.7; 2.0 is Python 3
  only. Legacy toolchains stuck on 1.x are effectively frozen[^4].

## When to Use / When Not

**Use when:**
- You want a `Makefile` replacement written in Python, with tasks as real
  functions.
- You already use (or plan to use) Fabric for deployment and want a consistent
  task/config model underneath.
- You want dependency-free, cross-platform command orchestration with a
  layered config system.

**Avoid when:**
- You need native parallel task execution or a true dependency DAG — reach for
  a build tool designed around that.
- You want the `click`/`argparse` ecosystem and its plugins for your CLI.
- Your project is a general-purpose CLI app rather than a set of dev/ops tasks —
  Invoke's parser is tuned for the task-runner shape, not arbitrary CLIs.

## Alternatives

- pyinvoke/fabric — use instead when the job is SSH-based remote execution and
  deployment; Fabric is Invoke plus a network layer.
- pallets/click — use instead when building a standalone CLI application rather
  than a task file; richer parsing and a large plugin ecosystem.
- casey/just — use instead when you want a language-agnostic, `make`-like command
  runner without writing Python.
- nox / tox — use instead when the core need is testing across multiple Python
  environments; both are session/environment runners, not general task runners.
- pydoit/doit — use instead when you need real file-dependency tracking and
  incremental "only rebuild what changed" behavior.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2012-02 | Extracted from Fabric's task/execution core[^1]. |
| 1.0.0 | 2018-05 | First stable release, shipped alongside Fabric 2.0[^1]. |
| 2.0.0 | 2023-01 | Python 3 only; dropped Python 2.7 support[^4]. |
| latest | 2026-04 | Ongoing maintenance on `main`; large open-issue backlog. |

## References

[^1]: Fabric — "Upgrading from 1.x" and project history, explaining Invoke as the
extracted task/execution core underlying Fabric 2+. https://www.fabfile.org/upgrading.html
[^2]: Jeff Forcier (bitprophet), project roadmap and maintenance notes.
https://bitprophet.org/projects#roadmap
[^3]: Invoke documentation and issue tracker on parallel/concurrent execution as a
roadmap item rather than a shipped feature. https://github.com/pyinvoke/invoke/issues
[^4]: Invoke changelog (Python version support across the 1.x → 2.0 transition).
https://pyinvoke.org/changelog.html

## Tags

python, task-runner, cli, build-tool, subprocess, automation, devops, make-alternative, fabric, command-execution
