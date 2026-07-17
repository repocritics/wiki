# gruns/icecream

> A print-debugging helper for Python: `ic(x)` prints the expression source, its value, and optionally the call site — then returns the value so it drops into existing code.

[GitHub repo](https://github.com/gruns/icecream) ·
[PyPI: icecream](https://pypi.org/project/icecream/) ·
[License: MIT](https://github.com/gruns/icecream/blob/master/LICENSE.txt)

## Overview

IceCream is a single-purpose developer tool: a better `print()` for the
throwaway debugging that most Python programmers do dozens of times a day.
Calling `ic(foo(123))` prints `ic| foo(123): 456` — both the literal source
of the argument and its evaluated value — so you never again type
`print("foo(123)", foo(123))` by hand. Called with no arguments, `ic()`
prints the filename, line number, and enclosing function, which turns it
into a lightweight execution tracer. Output goes to stderr, is pretty-printed
via `pprint`, and is syntax-highlighted.

The project was created in 2018 by Ansgar Grunseid (`gruns`)[^1] and became
the reference implementation for a family of `ic()` clones in other
languages (Rust, Go, C++, Ruby, Dart, and a dozen more, all linked from the
README). As of 2026 the repo has roughly 10k stars and is maintained by Ivan
Karabadzhak (`Jakeroid`) with support from Lunal[^2].

The defining tension is that IceCream's headline feature — showing you the
*source text* of what you passed — requires recovering that source at
runtime. That machinery is reliable in ordinary scripts but degrades in
REPLs, `exec`'d code, and frozen/optimized environments, and it adds
per-call overhead that makes `ic()` unsuitable for hot loops or production
logging. It is a development-time convenience, not infrastructure.

## Getting Started

```bash
pip install icecream
```

```python
from icecream import ic

def foo(i):
    return i + 333

ic(foo(123))          # ic| foo(123): 456

d = {'key': {1: 'one'}}
ic(d['key'][1])       # ic| d['key'][1]: 'one'

ic()                  # ic| example.py:11 in <module>   (no args → call site)
```

`ic()` returns its argument(s), so it can wrap an existing expression
without restructuring code: `b = half(ic(a))` prints `a` and still assigns
the result of `half(a)`.

## Architecture / How It Works

The interesting engineering is source retrieval. To print `foo(123)` as
text, `ic()` must, from inside its own call, find the exact AST node of the
call in the caller's source. IceCream delegates this to Alex Hall's
`executing` library[^3], which walks the calling frame's bytecode and maps
the current instruction back to a node in the parsed source tree. This is
the same technique that powers `snoop` and modern `pytest`/traceback
tooling, and it is far more robust than the naive `inspect.getsource` +
regex approach the project used in its earliest versions.

Once the argument nodes are located, IceCream:

- serializes each value with `argToStringFunction` (default
  `icecream.argumentToString`, which wraps `pprint.pformat`),
- prepends a `prefix` (default `ic| `),
- optionally prepends context (`filename:lineno in function`), and
- writes the assembled string to `outputFunction` (default: stderr).

All four are swappable through `ic.configureOutput(...)`. `argumentToString`
is a `functools.singledispatch` registry, so you can register per-type
formatters (e.g. summarize a NumPy array by shape/dtype instead of dumping
it) and unregister them later.

Syntax highlighting uses Pygments; colored output relies on `colorama` for
cross-platform ANSI support. The public surface is deliberately tiny: a
single global singleton `ic` (an instance of `IceCreamDebugger`), plus
`install()`/`uninstall()`, which add or remove `ic` from the `builtins`
module so it is callable in every file without an import.

## Production Notes

- **Not for production.** IceCream is a dev tool. The README itself ships a
  graceful-fallback import snippet so shipped code doesn't `ImportError` if
  the package is absent, and `ic.disable()` silences output while still
  returning arguments. But the per-call cost — frame inspection plus source
  mapping — is real; never leave `ic()` in a hot path.
- **Source must be available.** If `executing` cannot recover the call's
  source (interactive REPL sessions, `exec`/`eval`'d strings, some frozen
  PyInstaller-style builds, or `python -O`-stripped bytecode), IceCream
  falls back to printing values without the expression text. This is the
  single most common "why doesn't it show my variable name" surprise.
- **Global mutable state.** `ic` is a process-wide singleton and
  `configureOutput` mutates it globally. A library that calls
  `ic.configureOutput(...)` changes behavior for the whole process; prefer a
  local `IceCreamDebugger` instance if you need isolated configuration.
- **`install()` pollutes builtins.** Convenient for a top-level script,
  hazardous in a library — it makes `ic` a global name for every importing
  module and shadows anything else named `ic`. Reserve it for your own
  application entry points.
- **stderr, not logging.** Output bypasses the `logging` module. For log
  integration use `ic.format(...)` to get the string and pass it to your
  logger; the README notes this is "a bit clunky," and there is a
  long-standing open request for native log-level support[^4].
- **Ordering with stdout.** Because `ic()` writes to stderr and `print()` to
  stdout, interleaved output can appear out of order in some terminals and
  CI logs.

## When to Use / When Not

**Use when:**
- You reach for `print()` to inspect a value and want the expression label
  and pretty-printing for free.
- You want a quick execution trace (which branch ran, in what order) without
  setting up a debugger.
- You're wrapping a subexpression in place and need the return value
  preserved.

**Avoid when:**
- You need structured, leveled, or persisted logs — use `logging` or
  `loguru`.
- You're in a hot loop or latency-sensitive path — the source-inspection
  overhead matters.
- You're running where source is unavailable (embedded interpreters, frozen
  apps, `exec`'d code) and rely on seeing expression text.
- A one-off print suffices: Python 3.8+ f-strings already do `f"{x=}"`,
  which covers the simplest case with zero dependencies.

## Alternatives

- `alexmojaki/snoop` — line-by-line tracing decorator from the author of
  `executing`; use when you want a full trace of a function, not point
  probes.
- `cool-RR/PySnooper` — decorator that logs every line, variable change, and
  call; use for "why did this function do that" over a whole block.
- `Textualize/rich` — `rich.inspect()` / `rich.print()` for richer object
  and structure rendering; use when presentation matters more than showing
  the source expression.
- `Delgan/loguru` — batteries-included logging; use when the output should
  persist, have levels, or ship to a sink rather than being scratch output.
- Built-in `f"{x=}"` (Python 3.8+) and `breakpoint()`/`pdb` — use when you
  want zero dependencies or an interactive debugger instead of a print.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-02 | First release; `ic()` with expression + value printing[^1]. |
| 2.x | — | Dropped Python 2 support; Python 3 / PyPy3 only. |
| current | 2026-07 | Actively maintained; `singledispatch` `argumentToString`, `contextAbsPath`, `install()`/`uninstall()`. Latest push 2026-07-06[^5]. |

Precise per-release dates are not asserted here where the changelog is not
confidently known; the table reflects the repository's creation date and
current activity from the GitHub API.

## References

[^1]: gruns/icecream repository, created 2018-02-13. https://github.com/gruns/icecream
[^2]: icecream README — maintainer note (Ivan Karabadzhak / Jakeroid, with support from Lunal). https://github.com/gruns/icecream#readme
[^3]: `executing` by Alex Hall — locates the AST node of the current call from a stack frame. https://github.com/alexmojaki/executing
[^4]: icecream issue #146 — request for built-in log-level support. https://github.com/gruns/icecream/issues/146
[^5]: GitHub API `repos/gruns/icecream` — stars, forks, license, last push (fetched 2026-07-17).

## Tags

python, debugging, print-debugging, developer-tools, introspection, ast, cli, logging, pretty-print, pypy
