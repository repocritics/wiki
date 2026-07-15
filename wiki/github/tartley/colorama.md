# tartley/colorama

> Makes ANSI color/cursor escape sequences work on legacy Windows consoles — and does nothing everywhere else, by design.

[GitHub repo](https://github.com/tartley/colorama) ·
[PyPI](https://pypi.org/project/colorama/) ·
[License: BSD-3-Clause](https://github.com/tartley/colorama/blob/master/LICENSE.txt)

## Overview

Colorama is a small, single-purpose Python library: it lets programs that emit
ANSI escape sequences (colored text, cursor movement) produce correct output on
Windows terminals that don't natively understand ANSI. On Unix and macOS, where
terminals have handled ANSI for decades, Colorama deliberately does nothing[^1].
The whole library is a few hundred lines of code with no dependencies beyond the
standard library.

Its outsized importance is a supply-chain accident more than a feature story.
Because so many CLI tools want colored output that "just works" on Windows,
Colorama became a near-universal transitive dependency — it is pulled in by pip,
pytest, click-based tools, and thousands of others. That places a tiny,
low-churn library at the root of an enormous dependency graph, which is exactly
why its security posture and maintenance cadence matter far more than its ~3.8k
stars would suggest[^2].

The defining tension is scope discipline against a shrinking core use case. The
maintainers explicitly refuse pull requests that add ANSI *generation* (new
color helpers, styling shortcuts) — Colorama only *translates* existing ANSI
into Win32 API calls[^1]. Meanwhile, Windows 10+ ships native ANSI support, so
the emulation path that justified the library is needed by fewer users each year.
Colorama's answer (v0.4.6+) is `just_fix_windows_console()`, which prefers the
OS's native ANSI switch and only falls back to stream wrapping on old Windows.

## Getting Started

```bash
pip install colorama
# or: conda install -c anaconda colorama
```

```python
from colorama import just_fix_windows_console, Fore, Back, Style

# Enable ANSI on Windows if needed; no-op elsewhere. Safe to call repeatedly.
just_fix_windows_console()

print(Fore.RED + "some red text")
print(Back.GREEN + "and a green background")
print(Style.RESET_ALL + "back to normal now")
```

`Fore`, `Back`, and `Style` are plain string constants holding raw ANSI codes,
so you concatenate them into ordinary strings. `Style.RESET_ALL` clears
foreground, background, and brightness; Colorama also resets automatically at
program exit.

## Architecture / How It Works

Colorama has two public entry points with different histories:

- **`just_fix_windows_console()`** (v0.4.6+) — the recommended path. On recent
  Windows 10+ with a real console, it enables the OS's built-in ANSI support and
  leaves your streams untouched. On older Windows it wraps `sys.stdout` /
  `sys.stderr`. Everywhere else it is a no-op. It is idempotent and safe to call
  multiple times[^1].
- **`init(**kwargs)`** — the original interface, kept for backwards
  compatibility. More configurable (`autoreset`, `strip`, `convert`, `wrap`) but
  with more footguns: calling it more than once can produce nested wrappers and
  broken output.

The core machinery is `AnsiToWin32`, a proxy stream object. When wrapping is
active, it intercepts `.write()`, scans the byte stream for recognized ANSI CSI
sequences (`ESC [ <params> <command>`), and issues the equivalent
`SetConsoleTextAttribute` / cursor / clear calls via `ctypes.windll`. Only a
fixed subset of sequences is translated: SGR color/brightness codes 0–2/22/30–49,
cursor moves (`A`/`B`/`C`/`D`/`H`/`f`), and screen/line clears (`J`/`K`)[^1].
Any ANSI sequence Colorama recognizes but can't map is silently stripped on
Windows; unrecognized forms (non-CSI codes, alternative introducers) pass
through untouched.

Two behaviors are worth internalizing. First, `strip`: by default Colorama
removes ANSI codes when output is redirected (not a TTY) or on Windows without
conversion, so piping to a file doesn't leave escape gibberish. Second, on
Windows, Colorama does not emulate "dim" — `Style.DIM` renders the same as
normal brightness, because the Win32 console has no dim attribute.

## Production Notes

- **`init()` vs `just_fix_windows_console()`.** New code should use the latter.
  The old `init()` is not safe to call twice, and libraries that call it can
  fight application-level configuration. A library should generally not call
  either at import time; leave initialization to the application.
- **Stream wrapping breaks identity assumptions.** Because `init()`/wrapping
  replaces `sys.stdout`/`sys.stderr` with proxy objects, code that captures the
  original stream, checks `isinstance`, or relies on the file descriptor can be
  surprised. Use `init(wrap=False)` plus an explicit `AnsiToWin32(...).stream`
  when you need targeted conversion without global replacement.
- **Modern Windows often needs nothing.** On Windows 10+ terminals with native
  virtual-terminal processing (Windows Terminal, recent conhost), ANSI already
  works. `just_fix_windows_console()` short-circuits to the native switch, but
  many apps could drop the dependency entirely if they no longer target old
  Windows.
- **Supply-chain surface.** As a deep transitive dependency, Colorama has been a
  repeated typosquatting/impersonation target on PyPI (lookalike names). Pin from
  a trusted index and verify the package name is exactly `colorama`[^2].
- **Not a styling library.** It offers only rudimentary constants and will not
  grow more. For rich styling, tables, or markup, pair it with a dedicated
  library and let Colorama handle only the Windows-compatibility layer. It also
  registers an atexit reset; use `deinit()` / `reinit()` to control wrapping in
  long-lived host processes.

## When to Use / When Not

**Use when:**
- You ship a CLI that must show ANSI colors correctly on older Windows consoles.
- You already emit ANSI (directly or via termcolor/blessings) and want one call
  to make it portable to Windows.
- You want a zero-dependency, tiny, permissively licensed shim.

**Avoid when:**
- You only target Windows 10+/modern terminals — native ANSI likely suffices.
- You want a full styling/TUI toolkit (colors, spinners, tables, markup) — use
  a higher-level library and keep Colorama, if at all, only for the Win32 bridge.
- You are on Unix/macOS exclusively — Colorama is a no-op and adds nothing.

## Alternatives

- textualize/rich — full-featured console rendering (markup, tables, progress);
  use instead when you want styling, not just Windows ANSI compatibility.
- termcolor/termcolor — minimal ANSI color helpers; use with Colorama when you
  want simple color generation plus Windows support.
- erikrose/blessings (and blessed) — terminal capability/cursor library; use
  when you need curses-style control beyond color.
- click (its `style`/`echo`) — use when you're already building a Click CLI and
  want integrated cross-platform colored output.
- Native `os.system("")` / `colorama`-free virtual-terminal enabling — use when
  you target only modern Windows and can flip VT processing yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2010 | Initial release; Win32 ANSI translation shim[^3]. |
| 0.4.0 | 2018-05 | Modernization; broader ANSI/cursor handling. |
| 0.4.4 | 2020-10 | Maintenance release under Arnon Yaari co-maintenance. |
| 0.4.6 | 2022-10-25 | `just_fix_windows_console()` added; Python 2 support dropped[^1]. |

Copyright is held by Jonathan Hartley and Arnon Yaari; the project has been
BSD-3-Clause throughout[^1]. As of this writing the repository shows ~3,794
stars, 280 forks, and 137 open issues, with commits still landing in 2026 —
low-velocity but not abandoned[^2].

## References

[^1]: Colorama README and usage docs (tartley/colorama, master).
      https://github.com/tartley/colorama/blob/master/README.rst
[^2]: GitHub repository metadata (tartley/colorama), fetched via GitHub API.
      https://github.com/tartley/colorama
[^3]: PyPI release history for colorama.
      https://pypi.org/project/colorama/#history

## Tags

python, terminal, ansi, cli, windows, console, colors, cross-platform, text-styling, zero-dependency
