# p-ranav/indicators

> Header-only C++ library for terminal progress bars and spinners, styled after Python's tqdm.

[GitHub repo](https://github.com/p-ranav/indicators) ·
[License: MIT](https://github.com/p-ranav/indicators/blob/master/LICENSE)

## Overview

`indicators` is a header-only C++11 library for drawing activity indicators in a
terminal: determinate progress bars, indeterminate "bouncing" bars, unicode
block bars, spinners, and containers for managing several of them at once. It was
created in 2019 by Pranav Srinivas Kumar (p-ranav), the author of a cluster of
single-header C++ utility libraries — argparse, tabulate, csv2 — that share a
common design ethos: drop one header in, configure through named options, no
build-system integration required[^1].

The library's reference point is Python's tqdm[^2]. The time meter format
(`[{elapsed}<{remaining}]`), the tick-based update model, and the "wrap an
iterable" ergonomics are all borrowed from it. What indicators adds over a
naive `\r`-and-percentage loop is a typed configuration system, bundled ANSI
color handling, and cursor control that behaves across platforms.

The defining tension is scope. indicators renders indicators and nothing else.
It has no layout engine, no event loop, and no notion of the rest of your
terminal output — the moment your own code writes to `stdout` next to a live
bar, the display corrupts. Within its narrow lane it is pleasant; the moment you
want a composed TUI you have outgrown it and should reach for a full framework.

## Getting Started

Header-only. Copy `include/indicators/` into your project, or use the
amalgamated `single_include/indicators/indicators.hpp`. Requires a C++11
compiler; no linking, no dependencies beyond the standard library.

```cpp
#include <indicators/progress_bar.hpp>
#include <chrono>
#include <thread>

int main() {
  using namespace indicators;
  ProgressBar bar{
    option::BarWidth{50},
    option::Start{"["}, option::Fill{"="}, option::Lead{">"},
    option::Remainder{" "}, option::End{"]"},
    option::PostfixText{"Extracting"},
    option::ForegroundColor{Color::green},
    option::ShowPercentage{true},
    option::FontStyles{std::vector<FontStyle>{FontStyle::bold}}
  };

  while (!bar.is_completed()) {
    bar.tick();                                   // +1%
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
  }
}
```

`bar.tick()` advances by exactly 1%; `bar.set_progress(n)` jumps to a value in
`[0, 100]`. Options are also mutable at runtime via `bar.set_option(...)`.

## Architecture / How It Works

Every configuration knob is a distinct strongly-typed struct in the
`indicators::option` namespace — `BarWidth`, `ForegroundColor`, `PostfixText`,
and so on. Each wraps a value and carries a compile-time `Setting` id. A bar
stores these in a `std::tuple`, and lookups (`get_value<id>()`) resolve at
compile time through index metaprogramming. This is why the constructor accepts
options in any order and why passing an unsupported option to a given indicator
is a compile error rather than a silently ignored runtime flag. It is more
machinery than a struct of fields, and it is the library's central design
decision.

Color and styling are handled by a vendored copy of `termcolor` (ANSI escape
sequences) rather than an external dependency[^3]. Cursor manipulation —
hiding the cursor, moving up N lines to redraw stacked bars — lives in
`cursor_control.hpp` and `cursor_movement.hpp`, with separate code paths for
POSIX terminals and the Windows console API.

The indicator types are:

- **ProgressBar** — the workhorse; progress is a `size_t` in `[0, 100]`.
- **BlockProgressBar** — uses unicode eighth-block glyphs and a `float`
  fill for sub-character smoothness.
- **IndeterminateProgressBar** — a bouncing lead with no known total;
  you call `mark_as_completed()` when done.
- **ProgressSpinner** — cycles a user-supplied vector of spinner frames.
- **MultiProgress\<Indicator, N\>** — a fixed, compile-time count of bars;
  updates are addressed by template index (`bars.tick<0>()`).
- **DynamicProgress\<Indicator\>** — a runtime-growable container of
  `unique_ptr` bars; `push_back` returns an index.

Thread safety is scoped narrowly: an individual bar guards its own `tick` /
`set_progress` / redraw with a mutex, so several threads driving the *same* bar
is safe. It does not coordinate your other writes to `stdout`.

## Production Notes

**Windows needs setup.** ANSI escape sequences require enabling virtual
terminal processing, and unicode bars (`BlockProgressBar`, block glyphs) need a
UTF-8 code page (`chcp 65001`) and a font that has the glyphs. On older
`cmd.exe` / conhost you will see literal escape codes or mojibake. Modern
Windows Terminal is fine.

**Bars do not share the terminal.** There is no compositing layer. If your
application logs to `stdout` while a bar is live, the log line lands inside the
bar and both are mangled. Print your own output *before* starting the bar, route
logs to `stderr` (only if the bar is on `stdout`), or use `DynamicProgress`,
which owns its multi-line region. This is the single most common surprise.

**Resolution.** The plain `ProgressBar` percentage is an integer 0–100, so on a
wide bar the fill moves in visible steps. `BlockProgressBar` exists precisely to
get smooth motion; prefer it when the bar is wide and updates are frequent.

**Cursor state on abnormal exit.** If you hide the cursor
(`show_console_cursor(false)`) and your program throws or is killed before the
matching `show_console_cursor(true)`, the user is left with an invisible cursor.
Guard the restore with RAII or a signal handler for long-running tools.

**Compile cost and inclusion.** Header-only means the templates recompile in
every translation unit that includes them; in large builds prefer including the
bar headers in as few `.cpp` files as possible. The `single_include`
amalgamation is convenient for vendoring but does not reduce compile time.

**Maintenance velocity is low.** The API has been stable at the 2.x line for
years and the project sees infrequent commits (last push 2025-05-09 at time of
writing, ~49 open issues)[^4]. For a self-contained, unchanging display library
this is a feature more than a risk — there is little surface to break — but do
not expect rapid response to platform-specific issues.

## When to Use / When Not

**Use when:**
- You want a good-looking progress bar or spinner in a CLI with zero build
  integration and no dependencies.
- You need several concurrent bars (downloads, parallel jobs) and want the
  stacking handled for you.
- You are on C++11 and cannot pull in a heavier TUI framework.

**Avoid when:**
- You need a full terminal UI — panels, input, layout. Use a TUI framework.
- Your program interleaves arbitrary logging with progress output and you are
  not willing to segregate the streams.
- You need a built-in `for`-range/iterator wrapper that auto-counts totals the
  way tqdm does; here you drive the percentage yourself.

## Alternatives

- arthursonzogni/FTXUI — full C++ terminal UI framework with gauge/progress
  components; use it when you need layout, input, and composition, not just a bar.
- tqdm/tqdm — the Python original; use it when your tool is already Python and
  you want auto-counting over any iterable.
- prakhar1989/progress-cpp — a single-class header progress bar; use it when
  you want something minimal and are fine styling it yourself.
- gipert/progressbar — another small single-header bar; similar minimalist niche.
- p-ranav/tabulate — same author, for tabular terminal output rather than
  progress; complementary, not a replacement.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-12-03 | Repository created[^4]. |
| 2.x | — | Option/setting-based API (the current design); stable for years. |
| 2.3 | — | Latest tagged release per project version badge[^1]. |

Exact release dates for the 2.x tags are not restated here where they could not
be verified; consult the GitHub Releases page for authoritative timestamps.

## References

[^1]: p-ranav/indicators README and repository. https://github.com/p-ranav/indicators
[^2]: tqdm — Python progress bar, cited as the inspiration for the time-meter format. https://github.com/tqdm/tqdm
[^3]: termcolor — header-only ANSI color library, vendored into indicators. https://github.com/ikalnytskyi/termcolor
[^4]: GitHub repository metadata (creation date, last push, open issue count) via GitHub API, retrieved 2026-07. https://github.com/p-ranav/indicators

## Tags

cpp, cpp11, header-only, single-header, progress-bar, spinner, cli, terminal, tui, ansi-colors, mit-license
