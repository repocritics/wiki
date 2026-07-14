# pallets/click

> A Python toolkit for building composable command line interfaces from decorated functions.

[GitHub repo](https://github.com/pallets/click) ·
[Official website](https://click.palletsprojects.com) ·
[License: BSD-3-Clause](https://github.com/pallets/click/blob/main/LICENSE.txt)

## Overview

Click ("Command Line Interface Creation Kit") is a Python library for building CLIs by decorating functions with `@click.command`, `@click.option`, and `@click.argument`. It was written by Armin Ronacher (author of Flask) and released in 2014 as part of the Pallets suite, which also includes Flask, Jinja, and Werkzeug[^1]. Its design goal is arbitrary command nesting, automatic help-page generation, and lazy loading of subcommands, with sensible defaults out of the box[^2].

The defining tradeoff is **declarative decorators versus explicit control**. Click builds a tree of `Command`/`Group` objects from decorator metadata and takes over the whole program lifecycle: it parses arguments, converts types, prompts for missing values, prints help, and calls `sys.exit` on your behalf. This removes a large amount of boilerplate compared to the stdlib `argparse`, but it also means the framework owns the process. Embedding a Click command inside a larger program, capturing its return value, or overriding its exit behavior all require opting out of "standalone mode" — a recurring source of confusion.

Click sits at the base of a wider ecosystem: Flask's CLI (`flask run`) is built on it, and higher-level tools such as tiangolo/typer and the `rich-click` renderer wrap it rather than replace it. It is one of the most depended-upon packages on PyPI, which makes its parsing and completion behavior a de facto standard for Python CLI tooling.

## Getting Started

```bash
pip install click
```

```python
import click

@click.command()
@click.option("--count", default=1, help="Number of greetings.")
@click.option("--name", prompt="Your name", help="The person to greet.")
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for _ in range(count):
        click.echo(f"Hello, {name}!")

if __name__ == "__main__":
    hello()
```

Nesting subcommands under a group:

```python
@click.group()
@click.pass_context
def cli(ctx):
    ctx.ensure_object(dict)

@cli.command()
@click.argument("path")
@click.pass_context
def build(ctx, path):
    click.echo(f"building {path}")

# invoked as:  cli build ./src
```

## Architecture / How It Works

Click builds a static object tree at import time and walks it at call time:

1. **Decorators as builders** — `@click.command()` wraps a function in a `Command`; each `@click.option`/`@click.argument` attaches a `Parameter` to it. `@click.group()` produces a `Group` (a `Command` that dispatches to subcommands). The decorators stack, so the source order of options is preserved in help output.
2. **Context** — every invocation creates a `Context` object linked to its parent. `ctx.obj` carries user state down the tree; `ctx.ensure_object`, `pass_context`, and `make_pass_decorator` share it between a group and its subcommands. The context also manages cleanup (`ctx.call_on_close`, `with ctx:`), which is how Click closes file handles opened by `click.File` types.
3. **Parameters and types** — every option/argument has a `ParamType` (`INT`, `Path`, `Choice`, `File`, custom subclasses) that both converts and validates the raw string, producing uniform error messages. Options can be filled from environment variables (`envvar=` or a group-wide `auto_envvar_prefix`), prompts, or defaults.
4. **Parsing** — Click ships its own argument parser rather than delegating to `argparse`; historically it derived from `optparse` internals. It deliberately does **not** do `argparse`-style prefix abbreviation (`--ver` will not match `--verbose`), which trades convenience for stability.
5. **Dispatch and exit** — the outermost command runs in *standalone mode*: it invokes the callback, then catches `ClickException`/`Abort`, prints usage, and calls `sys.exit`. Callback return values are discarded unless you pass `standalone_mode=False` or use a `Group` with `chain=True` and a `result_callback`.

Cross-platform output goes through `click.echo` rather than `print`: it handles stream encoding, strips ANSI codes when the target is not a terminal, and initializes Colorama on Windows so `click.style`/`secho` colors work there.

## Production Notes

**Standalone mode surprises.** Because Click calls `sys.exit`, naively calling a command function from another Python function will terminate the interpreter. To embed a command or read its result, invoke it with `standalone_mode=False` (and handle `SystemExit`/`ClickException` yourself), or restructure so the callback logic lives in a plain function the command merely wraps.

**Return values are not chained by default.** A subcommand's return value is not passed to the group or to the next command. Command pipelines require `@click.group(chain=True)` plus a `result_callback`; this is not obvious from the basic examples and trips up people building Unix-style pipe commands.

**Shell completion changed in 8.0.** Click 8 replaced the older completion mechanism with a new system activated by eval-ing `_YOURPROG_COMPLETE=bash_source yourprog`, with separate scripts for bash, zsh, and fish[^3]. Completion set up against Click 7 does not carry over, and custom completion now uses `shell_complete=` callbacks instead of the old `autocompletion=`.

**Testing.** `click.testing.CliRunner` invokes a command in-process with a captured stdout/stderr and an isolated filesystem (`isolated_filesystem()`), returning a `Result` with `exit_code`, `output`, and any exception. This is the supported way to test CLIs; it avoids subprocess overhead but shares interpreter state, so global mutation between tests can leak.

**Encoding and `echo`.** Prefer `click.echo` over `print` in library code: it degrades gracefully on closed/piped streams and handles bytes vs text. Direct `print` of non-ASCII can still raise on misconfigured Windows consoles that `echo` papers over.

**Python-version churn.** Click 8.0 (2021) dropped Python 2 and set a Python 3 floor; later 8.x releases have continued to raise the minimum Python version and modernize type hints, so pinning `click>=8` without testing your interpreter version can pull in a release that no longer supports it. Check the changelog before bumping across minor 8.x versions[^4].

## When to Use / When Not

**Use when:**
- You want nested subcommands, grouped help, and prompts without hand-rolling a parser.
- You're in the Flask/Pallets ecosystem, or building on Typer/rich-click, which assume Click underneath.
- You need consistent type conversion, env-var binding, and testable commands via `CliRunner`.
- You're shipping a multi-command tool (`tool build`, `tool deploy`) rather than a single flat script.

**Avoid when:**
- You want zero third-party dependencies — the stdlib `argparse` covers simple cases with nothing to install.
- You prefer deriving the CLI from type hints — reach for Typer, which wraps Click and removes most decorators.
- You need the parser's return value in-process and don't want to fight standalone mode for a trivial script.
- Startup latency matters at extreme scale and importing Click's decorators is measurable — rare, but real for tiny hot-path scripts.

## Alternatives

- tiangolo/typer — use when you'd rather define options via Python type hints; it is built on Click and interoperates with it.
- google/python-fire — use when you want to expose an existing object or function tree as a CLI with essentially no CLI-specific code.
- docopt/docopt — use when you prefer to define the interface in the help/usage string itself rather than in decorators.
- python-poetry/cleo — use when you want class-based command objects and are already in the Poetry ecosystem.
- argparse (Python stdlib) — use when a single command with a few flags doesn't justify a dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-05 | Initial release by Armin Ronacher as part of Pallets[^1]. |
| 6.0 | 2015-08 | Consolidated API, wide adoption via Flask CLI. |
| 7.0 | 2018-09 | Large release; renamed/deprecated older APIs, improved Windows support[^4]. |
| 7.1 | 2020-03 | Final 7.x line; bug fixes, Python 3.8 support. |
| 8.0 | 2021-05 | Dropped Python 2, new shell-completion system, `shell_complete=` callbacks[^3]. |
| 8.1 | 2022-03 | Removed the `distutils`/setuptools runtime dependency; typing improvements. |
| 8.2 | 2025 | Raised the minimum Python version and reworked internal parser/typing[^4]. |

## References

[^1]: Pallets organization — Click and sibling projects (Flask, Jinja, Werkzeug). https://palletsprojects.com/
[^2]: Click documentation — "Why Click?" and project overview. https://click.palletsprojects.com/
[^3]: Click docs — Shell Completion. https://click.palletsprojects.com/en/stable/shell-completion/
[^4]: Click changelog / release notes. https://click.palletsprojects.com/en/stable/changes/

## Tags

python, cli, command-line, argument-parsing, decorators, pallets, developer-tools, terminal, library, bsd-licensed
