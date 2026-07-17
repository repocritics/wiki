# borntyping/python-colorlog

> A drop-in colored formatter for Python's standard `logging` module — ANSI colors keyed to log level, nothing more.

[GitHub repo](https://github.com/borntyping/python-colorlog) ·
[Package on PyPI](https://pypi.org/project/colorlog/) ·
[License: MIT](https://github.com/borntyping/python-colorlog/blob/main/LICENSE)

## Overview

colorlog is a single-purpose library: it provides a `ColoredFormatter` that
subclasses `logging.Formatter` and injects ANSI escape codes into log output
based on each record's level. It does not replace the stdlib logging system,
introduce a new logger, or restructure how you emit logs — you keep using
`logging.getLogger()`, handlers, and format strings exactly as before, and only
swap the formatter. That narrow scope is the whole value proposition: it is the
least invasive way to get red errors and yellow warnings in a terminal.

The project dates to 2012 and has spent most of its life supporting an unusually
wide range of Python versions, including Python 2[^1]. That backwards-compat
burden is the defining tension of the codebase: the maintainer has repeatedly
stated it makes the library hard to extend, and colorlog is now explicitly in
maintenance mode — bugfixes are published, but features that risk breaking
existing users are declined[^2]. For a formatter this stable, "done" is a
reasonable state, and the ~965-star, low-open-issue profile reflects a library
that most users install once and never think about again.

The 6.x line requires Python 3.6 or higher. 5.x is an interim release that warns
Python 2 users to downgrade, and 4.x is the final Python 2-compatible branch
that receives only essential bugfixes[^2].

## Getting Started

```bash
pip install colorlog
```

```python
import colorlog

handler = colorlog.StreamHandler()
handler.setFormatter(colorlog.ColoredFormatter(
    "%(log_color)s%(levelname)-8s%(reset)s %(blue)s%(message)s",
    log_colors={
        "DEBUG": "cyan",
        "INFO": "green",
        "WARNING": "yellow",
        "ERROR": "red",
        "CRITICAL": "red,bg_white",
    },
))

logger = colorlog.getLogger("example")
logger.addHandler(handler)
logger.warning("this line is yellow")
```

`colorlog.getLogger` and `colorlog.StreamHandler` are thin conveniences; you can
equally use `logging.getLogger` and `logging.StreamHandler` and only take
`ColoredFormatter` from the package.

## Architecture / How It Works

The entire mechanism is a format-string extension. `ColoredFormatter` recognizes
extra fields that the stdlib formatter does not: `%(log_color)s` expands to the
ANSI code for the current record's level, `%(reset)s` clears formatting, and
named codes like `%(blue)s`, `%(bg_white)s`, `%(bold_red)s` insert specific
colors. At format time the formatter builds a per-record dictionary of these
escape codes and hands it to the normal `%`-style (or `{`/`$`-style) substitution
machinery. Colors are therefore chosen entirely by which fields you put in the
format string — there is no separate rendering layer.

Two color-mapping arguments drive behavior. `log_colors` maps level names to
color names (falling back to `colorlog.default_log_colors`). `secondary_log_colors`
defines additional level-keyed palettes exposed as `%(<name>_log_color)s` — for
example a `message` secondary palette gives you `%(message_log_color)s` so the
message body can be colored independently of the level label. Color names cover
the eight standard ANSI colors, non-standard "bright"/`light_*` variants (terminal
support varies), and raw 256-color integers via `fg_196` / `bg_42`.

TTY detection and color suppression are handled inline. If a `stream` is passed,
the formatter checks whether it is a TTY and disables color on non-terminals
unless `force_color` is set. It also honors the `NO_COLOR` and `FORCE_COLOR`
environment variables (and matching `no_color` / `force_color` constructor args),
following the informal `NO_COLOR` convention[^3]. On Windows, `colorama` is
pulled in as a dependency and auto-initialized so that ANSI codes are translated
for the Windows console[^2].

Because it is just a `logging.Formatter` subclass, it composes with everything in
the stdlib config surface: `dictConfig` (via the `'()': 'colorlog.ColoredFormatter'`
factory syntax), `fileConfig` INI files (`class=colorlog.ColoredFormatter`), and
custom levels registered with `logging.addLevelName`.

## Production Notes

- **Color leaks into non-terminal sinks.** The TTY check only fires when you pass
  the target `stream` to the formatter and the handler actually writes to that
  stream. Misconfigure it — or send colored records to a file, journald, or a log
  aggregator — and you get raw `\x1b[` escape sequences in your logs. In
  production, gate coloring on `sys.stderr.isatty()` yourself, or rely on
  `NO_COLOR` in your deploy environment, and keep a separate uncolored formatter
  for file handlers.
- **`reset=True` matters.** By default the formatter appends a reset code so color
  does not bleed into subsequent terminal output. If you build format strings that
  set color after the message, or disable `reset`, a crashing record can leave the
  terminal in a colored state.
- **Windows depends on colorama.** Colored output on the legacy Windows console
  works only because colorama is installed and initialized. In constrained or
  vendored environments verify the transitive dependency is present.
- **Maintenance mode is real.** Do not build a product on the expectation of new
  features, new style options, or structured-output support. Feature PRs that
  affect backwards compatibility are declined by policy[^2]. This is a stability
  guarantee, not neglect — but plan around it.
- **It only colors; it does not structure.** colorlog produces human-readable
  terminal text. It has no notion of JSON, key/value context, or machine-parseable
  fields. If your log pipeline needs both pretty local output and structured
  production output, colorlog covers only the first half.

## When to Use / When Not

**Use when:**
- You already use stdlib `logging` and want colored level output with a one-line
  formatter swap.
- You want to keep `dictConfig`/`fileConfig`-based configuration unchanged.
- You need a tiny, stable, dependency-light addition (colorama only on Windows).

**Avoid when:**
- You want structured/JSON logging or bound context — reach for structlog.
- You want a batteries-included logging experience with rich tracebacks and
  rendering — loguru or rich cover that.
- You need actively evolving features; colorlog is deliberately frozen.

## Alternatives

- `hynek/structlog` — use instead when you need structured, context-bound logs
  with pluggable rendering (colored dev output *and* JSON in production).
- `Textualize/rich` — use `rich.logging.RichHandler` instead when you want colored
  logs plus rich tracebacks, tables, and markup in the terminal.
- `Delgan/loguru` — use instead when you want a batteries-included logger that
  replaces stdlib boilerplate rather than extending it.
- `borntyping/jsonlog` — the maintainer's own JSON formatter; use when you want
  machine-readable stdlib logging output.
- `chris1610/coloredlogs` (coloredlogs) — use instead for a similar
  color-the-stdlib approach with built-in field styling and install helpers.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-09 | First release as a colored `logging` formatter[^1]. |
| 4.x | — | Final line supporting Python 2. Bugfix-only. |
| 5.x | — | Interim release; warns Python 2 users to downgrade. |
| 6.0 | 2021 | Requires Python 3.6+; backwards-compat break to ease maintenance[^2]. |
| 6.9.0 | 2024 | Latest 6.x maintenance release at time of writing[^4]. |

## References

[^1]: Repository created 2012-09-05; GitHub repository metadata,
https://github.com/borntyping/python-colorlog
[^2]: colorlog README, "Status" and installation sections (Python version
support, maintenance-mode policy, Windows/colorama behavior),
https://github.com/borntyping/python-colorlog/blob/main/README.md
[^3]: NO_COLOR informal standard, https://no-color.org/
[^4]: colorlog release history on PyPI,
https://pypi.org/project/colorlog/#history

## Tags

python, logging, terminal, ansi-colors, cli, formatter, stdlib, maintenance-mode, colorama, developer-tools
