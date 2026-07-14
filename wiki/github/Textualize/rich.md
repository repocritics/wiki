# Textualize/rich

> A Python library for rich text and formatting in the terminal — color, tables, progress bars, markdown, syntax highlighting, and tracebacks, from one API.

[GitHub repo](https://github.com/Textualize/rich) ·
[Documentation](https://rich.readthedocs.io/en/latest/) ·
[License: MIT](https://github.com/Textualize/rich/blob/master/LICENSE)

## Overview

Rich is a terminal-rendering library written by Will McGugan and first
released to PyPI in early 2020[^1]. It occupies the middle layer of the
Python CLI stack: above raw ANSI escape codes and `colorama`, below full
TUI frameworks. You hand it renderables — strings with markup, `Table`,
`Panel`, `Progress`, `Syntax`, `Markdown`, `Tree` — and it measures the
terminal, word-wraps, and emits styled output. It is a rendering library,
not an application framework: there is no event loop, no widgets, no input
handling. That scope discipline is why it became a near-ubiquitous
dependency — `pip` uses it for progress bars, Typer builds its output on
it, and countless internal CLIs pull it in.

The project is mature and widely deployed (~57k stars, ~2.2k forks). It is
still maintained — the most recent release, 15.0.0, shipped in April 2026,
and commits continued through mid-2026[^2] — but the pace has slowed from
its 2020–2022 peak as its author's attention moved to the sibling project
Textual. Treat Rich as feature-complete infrastructure: stable, low-churn,
and unlikely to surprise you across upgrades.

The defining tension is that Rich looks effortless in an interactive
terminal and behaves differently the moment output is not a TTY. Terminal
width, color depth, and whether animation runs at all are all detected at
runtime, so the same code produces different bytes in your shell, in a CI
log, in a Docker container, and in a piped file. Most Rich "bugs" filed
against downstream tools are this detection surprise, not defects.

## Getting Started

```sh
python -m pip install rich
python -m rich   # render a capability test card to your terminal
```

```python
from rich.console import Console
from rich.table import Table

console = Console()
console.print("Hello, [bold magenta]World[/]!", ":vampire:")

table = Table(title="Releases")
table.add_column("Version", style="cyan", no_wrap=True)
table.add_column("Date", justify="right", style="green")
table.add_row("15.0.0", "2026-04-12")
table.add_row("14.0.0", "2025-03-30")
console.print(table)
```

The `[bold magenta]...[/]` console markup is BBCode-like and is Rich's own
mini-language, distinct from the `style=` keyword; the two can be mixed.

## Architecture / How It Works

Everything routes through the `Console` object. On construction it probes
the environment — `isatty()`, `COLUMNS`/`LINES`, `TERM`, `NO_COLOR`,
`FORCE_COLOR`, Windows console version, Jupyter — and derives a color
system (standard 16, 256, or truecolor) and a width. `console.print()`
takes any object implementing the **Console Protocol** (`__rich__` or
`__rich_console__`), asks it to yield `Segment`s (a run of text plus a
`Style`), measures them, wraps to the target width, and writes ANSI.

Renderables compose recursively: a `Table` cell may contain a `Panel` that
contains `Markdown`, and each is measured through the same `Measurement`
pass. This is what makes nesting "just work," and also where cost lives —
layout is computed on every render, so redrawing large renderables in a
tight loop is expensive.

Live-updating output (`Progress`, `Status`, `Live`) works by taking control
of the cursor: it renders a region, moves the cursor up, and overwrites on
each refresh via a background thread. Only **one** `Live` display can own
the screen region at a time; nesting or running two concurrently corrupts
the output. Any writes to the underlying `stdout` that bypass the console
during a `Live` session also break it — Rich provides `console.print`
inside the live region, or `Progress.console`, precisely to funnel writes.

Syntax highlighting delegates to Pygments[^3]; Rich supplies the terminal
theming and truecolor mapping. Markdown rendering uses `markdown-it-py`.
These are the only non-trivial runtime dependencies, keeping the install
light.

## Production Notes

**Non-TTY behavior is the number-one footgun.** When stdout is not a
terminal (piped, redirected, most CI runners, some container logs), Rich
disables color, disables animation (progress bars render as periodic full
lines, not in-place updates), and — critically — falls back to an assumed
width of 80 columns. Wide tables get truncated or wrapped in CI logs that
would look fine locally. Fixes: `Console(width=N)`, `force_terminal=True`,
`force_jupyter=`, or the `FORCE_COLOR` / `COLUMNS` environment variables.
Set these explicitly in CI rather than debugging "why does it look wrong."

**Markup injection.** `console.print(user_string)` interprets `[...]` in
the input as markup. A user value containing `[/]` or `[red]` will be
swallowed or raise. For any untrusted or arbitrary string, pass
`markup=False` or wrap with `rich.markup.escape()`. This is a correctness
and mild-security concern in log lines that echo user input.

**Performance.** Rich is not a high-throughput logging backend. Each render
recomputes layout; the `RichHandler` for the stdlib `logging` module adds
measurable per-record overhead versus a plain formatter. It is excellent
for human-facing output and debugging, poor for structured logs emitted at
thousands per second. Use a plain handler for hot log paths and reserve
`RichHandler` for interactive/dev use.

**Windows and glyph width.** Modern Windows Terminal gets truecolor and
emoji; legacy `cmd.exe` is capped at 16 colors and renders many glyphs as
boxes — the ceiling is the console, not the library. Separately, East-Asian
wide characters and emoji occupy two cells; fonts that render a glyph at a
width other than Unicode declares will misalign tables, which Rich cannot
fully correct.

**Upgrades** have been low-drama. Rich has kept a stable public API across
majors; most breaking changes are in rarely-touched internals. Pinning to a
major (`rich>=13,<16`) is safe for most consumers.

## When to Use / When Not

**Use when:**
- You want styled, human-readable CLI output — tables, trees, panels,
  progress — without hand-writing ANSI.
- You need pretty tracebacks or an improved logging handler during
  development and debugging.
- You want Markdown or syntax-highlighted code rendered in a terminal.
- You are already pulling it in transitively and want to use it directly.

**Avoid when:**
- You need an interactive, stateful TUI with widgets, focus, and input —
  reach for Textual instead; Rich has no event loop.
- You are emitting machine-readable or very high-volume logs — the layout
  cost and ANSI are liabilities, not features.
- Your only need is basic cross-platform color — `colorama` is far lighter.
- Output is always non-interactive (batch/CI) and you will not configure
  width/color — you inherit the detection surprises for little gain.

## Alternatives

- Textualize/textual — use instead when you need a full interactive TUI
  application (widgets, layout, input), not just formatted output; it is
  the same author's framework and builds on Rich's rendering ideas.
- tqdm/tqdm — use instead when you only need a progress bar and want a
  minimal, battle-tested dependency without the rest of Rich.
- tartley/colorama — use instead when all you need is portable ANSI color
  (notably Windows enablement) with a tiny footprint.
- prompt-toolkit/python-prompt-toolkit — use instead when you need
  interactive line editing, autocompletion, or a REPL rather than styled
  output.
- Textualize/rich-cli — use instead when you want Rich's rendering from the
  shell (`rich file.md`, `rich data.csv`) without writing Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.3.2 | 2020-01-26 | Earliest tagged release; project public.[^1] |
| 1.0.0 | 2020-05-03 | First stable line. |
| 10.0.0 | 2021-03-27 | Console protocol maturity, broad renderable set. |
| 11.0.0 | 2022-01-09 | Major release. |
| 12.0.0 | 2022-03-10 | Major release. |
| 13.0.0 | 2022-12-30 | Markdown engine moved to markdown-it-py era. |
| 14.0.0 | 2025-03-30 | Major release; Python support baseline advanced. |
| 15.0.0 | 2026-04-12 | Latest major as of this writing.[^2] |

## References

[^1]: Rich release history on PyPI/GitHub; earliest tags date to Jan–May
2020. https://github.com/Textualize/rich/releases
[^2]: Repository metadata (GitHub API): 15.0.0 released 2026-04-12, latest
push 2026-06-23. https://github.com/Textualize/rich
[^3]: Rich syntax highlighting is built on Pygments.
https://rich.readthedocs.io/en/latest/syntax.html

## Tags

python, terminal, cli, tui, console, formatting, syntax-highlighting, progress-bar, tables, ansi-colors, rendering, library
