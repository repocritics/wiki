# fastapi/typer

> A type-hint-driven layer over Click for building Python CLIs — declare parameters as function annotations, get parsing, help, and shell completion for free.

[GitHub repo](https://github.com/fastapi/typer) ·
[Official website](https://typer.tiangolo.com/) ·
[License: MIT](https://github.com/fastapi/typer/blob/master/LICENSE)

## Overview

Typer is a CLI-building library by Sebastián Ramírez (tiangolo), the author of FastAPI, and it applies the same idea to the terminal: you write a plain Python function, annotate its parameters with standard type hints, and the library introspects those hints to build the command-line interface[^1]. Positional parameters become CLI arguments, parameters with defaults become options, `bool` becomes a `--flag/--no-flag` pair, and `Enum` becomes a choice set. The project moved from tiangolo's personal namespace into the `fastapi` GitHub organization; the old `tiangolo/typer` URL redirects here.

Underneath, Typer is a wrapper over Click, the mature Pallets CLI toolkit. That inheritance is the defining tension of the project. Typer gives you Click's parsing, nesting, and completion machinery with far less boilerplate, but it also means you are one abstraction removed from the thing actually doing the work — and when you need Click's full surface you reach through Typer to the underlying command object.

Typer is still a pre-1.0 (0.x) library after more than six years, which is an honest signal about its API-stability posture: it is widely used in production but reserves the right to make breaking changes between minor releases, and it does. Development is active — the repository was pushed to on 2026-07-13[^5] — and the library is the near-default choice for new Python CLIs where the team already writes typed code.

## Getting Started

```bash
pip install typer
# pulls in rich (formatted output), shellingham (shell detection),
# and colorama on Windows
```

```python
# main.py — the modern Annotated style (recommended since 0.9.0)
import typer
from typing_extensions import Annotated

app = typer.Typer()


@app.command()
def hello(name: str, formal: Annotated[bool, typer.Option()] = False):
    if formal:
        print(f"Good day, Ms. {name}.")
    else:
        print(f"Hello {name}")


if __name__ == "__main__":
    app()
```

```console
$ python main.py hello Camila --formal
Good day, Ms. Camila.
```

Typer also ships a `typer` command that runs an arbitrary Python script as a CLI, even one that never imports Typer[^1] — useful for turning throwaway scripts into completion-enabled commands.

## Architecture / How It Works

At its core Typer reflects over a function's signature with `inspect` and the `typing` module, then constructs Click `Command` and `Group` objects from what it finds. The mapping is convention-driven:

- Parameters **without** a default → Click **arguments** (positional, required).
- Parameters **with** a default (or a `typer.Option(...)` marker) → Click **options**.
- `bool` → an automatic `--name/--no-name` toggle.
- `Enum`, `Path`, `datetime`, `UUID`, and file types → Click's built-in parsers.

There are two ways to declare parameter metadata, and this is the single most important thing to understand about the library. The **legacy default-value style** puts the marker in the default slot: `name: str = typer.Option("world")`. The **Annotated style**, recommended since 0.9.0[^2] and mirroring a parallel shift in FastAPI, moves the marker into the type: `name: Annotated[str, typer.Option()] = "world"`. The Annotated form keeps the real Python default in the default slot, which matters for testing and for calling the function directly (see Production Notes).

Rich integration is automatic: when `rich` is installed, help text and error panels are rendered with boxes and color[^1]. This can be disabled globally with the `TYPER_USE_RICH=0` environment variable or per-app via `rich_markup_mode`.

The Click relationship changed materially in **0.26.0**, when Typer **vendored** Click — copying Click's source into the package instead of depending on it externally — to unify the two codebases and stop tracking upstream Click's release cadence[^3]. The tradeoff stated by the maintainers is explicit: some Click functionality will not remain available as Typer's internal fork diverges. Separately, the `typer-slim` distribution (Typer without `rich`/`shellingham`) was retired in **0.22.0**; installing `typer-slim` now just installs full Typer[^3].

## Production Notes

**The default-value footgun.** With the legacy style, `def main(name: str = typer.Option(...))` sets the parameter's default to a Typer `OptionInfo` sentinel, not a usable value. Call that function directly in a unit test and `name` is an `OptionInfo` object, not a string. The fixes are to use `typer.testing.CliRunner` to invoke through the CLI machinery, or to adopt the Annotated style so the real default lives where Python expects it.

**Startup latency.** Importing `rich` (and Typer's own reflection at app construction) adds measurable process startup cost — typically on the order of 100 ms or more versus a bare `argparse` script. For CLIs invoked in tight loops or latency-sensitive tooling this is noticeable; disabling Rich (`TYPER_USE_RICH=0`) trims part of it, but the dependency footprint remains larger than stdlib.

**No native async.** Commands are ordinary synchronous functions. Async commands require wrapping the body in `asyncio.run(...)` yourself; there is no first-class `async def` command support.

**Click interop is one-directional.** Because Click is vendored, third-party Click plugins and extensions that expect the external `click` package will not automatically compose with a Typer app. When you need the raw Click object (for a plugin, a custom group class, or `click`-native tooling), reach through with `typer.main.get_command(app)`.

**0.x churn.** Being pre-1.0, minor releases have shipped behavior changes — the Click vendoring, the `typer-slim` removal, and the Annotated-style migration all landed as 0.x changes. Pin the version and read release notes before upgrading; do not assume patch-level safety across minors.

**Help-text parsing is brittle.** Rich formats help output into boxed panels; any tooling or test that scrapes `--help` text should either disable Rich or assert against the Click-native output, since the decorative rendering changes with Rich versions.

## When to Use / When Not

**Use when:**
- You already write typed Python and want the CLI to fall out of the type hints.
- You want automatic `--help`, shell completion (Bash/Zsh/Fish/PowerShell), and Rich-formatted errors without wiring them up.
- You're building anything from a two-line script to a deep tree of subcommands.
- Your team is in the FastAPI ecosystem and wants a consistent declaration style.

**Avoid when:**
- You need minimal dependencies or the fastest possible cold start — stdlib `argparse` has zero deps and lower import cost.
- You depend on the external Click plugin ecosystem, which no longer composes cleanly post-vendoring.
- You need an async-first command model or a guaranteed-stable 1.0 API surface.

## Alternatives

- pallets/click — the toolkit Typer is built on; use it directly when you want full control, the plugin ecosystem, or a mature stable API without a reflection layer.
- python/cpython (argparse, stdlib) — use when you cannot add third-party dependencies or need the smallest, fastest-starting CLI.
- google/python-fire — auto-generates a CLI from any object via introspection; use for quick, exploratory tools rather than curated interfaces.
- BrianPugh/cyclopts — a newer type-hint-driven CLI library positioned as a Typer alternative; use when you want the annotation style with fewer legacy patterns.
- docopt/docopt — defines the interface from a usage docstring; use when you'd rather write the help text and derive the parser from it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.x | 2019-12 | Initial release; thin type-hint layer over Click.[^5] |
| 0.9.0 | 2023-03 | `Annotated` parameter style introduced and recommended.[^2] |
| 0.22.0 | 2024 | `typer-slim` retired; it now installs full Typer.[^3] |
| 0.26.0 | 2025 | Click vendored into the package; interop caveats begin.[^3] |

## References

[^1]: Typer README and documentation. https://typer.tiangolo.com/
[^2]: Typer docs, "Annotations" / parameter declaration styles. https://typer.tiangolo.com/tutorial/options/
[^3]: Typer README — "Click code" and "typer-slim" sections (vendoring in 0.26.0, slim retirement in 0.22.0). https://github.com/fastapi/typer
[^4]: Click (Pallets) documentation. https://click.palletsprojects.com/
[^5]: GitHub API, repos/fastapi/typer — created 2019-12-24, 19.75k stars, MIT, last push 2026-07-13. https://github.com/fastapi/typer

## Tags

python, cli, command-line, type-hints, click, terminal, shell-completion, argument-parsing, fastapi-ecosystem, developer-tools
