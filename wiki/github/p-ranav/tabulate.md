# p-ranav/tabulate

> Header-only C++ library for rendering styled text tables to the terminal, with a fluent formatting API.

[GitHub repo](https://github.com/p-ranav/tabulate) ·
[License: MIT](https://github.com/p-ranav/tabulate/blob/master/LICENSE)

## Overview

`tabulate` is a single-header (or header-only) C++ library for building formatted
tables and printing them to a terminal or file. It is one of several widely used
single-header libraries from Pranav Srinivas Kumar (`p-ranav`), alongside
`argparse`, `indicators`, and `csv2`. The project targets `C++11` and up, though
the README examples assume `C++17`[^1]. It is MIT-licensed and has accumulated
around 2.2k stars and 155 forks since its first release in December 2019[^2].

The library's value proposition is ergonomic ANSI-styled output: you construct a
`Table`, add rows of strings (or nested tables), and drive appearance through a
fluent `.format()` interface — borders, corners, padding, alignment, font styles,
and 8-color foreground/background (via the bundled `termcolor`). Styling follows a
cell → row → column → table inheritance model, so you set defaults broadly and
override narrowly[^1].

The defining tradeoff is scope. `tabulate` is a *presentation* library, not a data
layer: there is no CSV/JSON ingest, no sorting, no paging, and no automatic
TTY detection. You assemble tables cell-by-cell in code, which is pleasant for a
dozen rows of CLI output and tedious for anything data-driven. Development has been
quiet since 2025 — the code is best read as effectively complete rather than
actively evolving[^2].

## Getting Started

There is nothing to build or link. Add `include/` to your include path, or drop the
amalgamated header from `single_include/` into your project.

```cpp
#include <tabulate/table.hpp>
using namespace tabulate;

int main() {
  Table movies;
  movies.add_row({"S/N", "Movie Name", "Director", "Budget", "Release Date"});
  movies.add_row({"tt1979376", "Toy Story 4", "Josh Cooley", "$200M", "21 Jun 2019"});
  movies.add_row({"tt3263904", "Sully", "Clint Eastwood", "$60M", "9 Sep 2016"});

  // Right-align the budget column
  movies.column(3).format().font_align(FontAlign::right);

  // Style the header row
  for (auto& cell : movies[0]) {
    cell.format()
      .font_color(Color::yellow)
      .font_align(FontAlign::center)
      .font_style({FontStyle::bold});
  }

  std::cout << movies << std::endl;   // or movies.print(std::cout)
}
```

Compile with any C++11+ compiler; no flags beyond the include path are required.

## Architecture / How It Works

The core object graph is `Table` → `Row` → `Cell`, with `Column` as a cross-cutting
view over the same cells. `add_row` accepts either a `std::vector<std::string>` (or
a `RowStream` for typed stream-insertion) or a nested `tabulate::Table`, which is how
nested tables and diagrams are composed. Indexing is `table[row][col]`, and
`table.column(i)` returns a column view; each of these exposes `.format()` returning
a `Format` object with a chainable, fluent setter for every visual property.

Rendering resolves each cell's effective style through the inheritance chain: an
explicit cell setting wins, otherwise the row's, then the column's, then the table's,
then library defaults[^1]. Widths are computed from content unless overridden with
`.width(n)`, and word-wrapping is automatic — a cell wraps at word boundaries unless
it contains explicit `\n`, in which case the embedded newlines are honored verbatim.
Each wrapped line is trimmed of surrounding whitespace.

Color and font styling emit raw ANSI escape sequences, sourced from a vendored copy
of `ikalnytskyi/termcolor`. Supported colors are the 8 classic ANSI colors; font
styles include `bold`, `italic`, `underline`, `blink`, `reverse`, `crossed`, and
others, whose actual rendering depends on the terminal[^1]. Multi-byte width is
handled on \*nix via `wcswidth`, gated behind an opt-in `.multi_byte_characters(true)`
so the more expensive width math is only paid when needed.

Beyond `operator<<`, the library ships exporters — Markdown and AsciiDoc — that
serialize a table to those text formats instead of ANSI-styled terminal output[^1].

## Production Notes

- **No TTY detection.** ANSI codes are emitted whenever colors/styles are set,
  regardless of whether the stream is a terminal. Piping `operator<<` output to a
  file or a non-TTY embeds escape sequences. For file output, use the Markdown or
  AsciiDoc exporters, or leave styling unset.
- **Multi-byte alignment depends on the locale.** Column width for CJK/runic/Arabic
  text uses `wcswidth`, which requires the system locale to be set (`setlocale`) and
  the relevant locale to exist. The README notes macOS lacking some locales (e.g.
  Arabic) causes misalignment — a platform limitation, not a bug you can fix in
  application code[^1].
- **Whitespace is trimmed per line.** Both auto-wrapped and manually-wrapped lines
  are trimmed on both sides, so intentional leading/trailing spaces inside a cell
  are dropped. Pad via `.padding_left/right` or width instead.
- **`C++11` floor, `C++17` examples.** The library compiles at C++11, but most
  documentation and samples assume C++17; porting the examples down may require
  adjustments.
- **Presentation only.** No input parsing, sorting, filtering, or streaming render.
  Large or dynamic datasets mean a lot of `add_row` boilerplate; pair with a
  separate CSV/JSON layer (the same author's `csv2`, for instance).
- **Maintenance cadence.** Last significant activity was mid-2025, with a few dozen
  open issues[^2]. Treat it as a stable, feature-complete dependency; do not expect
  rapid fixes or new features. Being header-only, vendoring a pinned copy is low-risk.

## When to Use / When Not

**Use when:**
- You want good-looking CLI tables (borders, colors, alignment) with minimal setup.
- You value a zero-build, header-only or single-header drop-in with no link step.
- Your table is authored in code and modest in size (config dumps, reports, diagrams).
- You need nested tables or ASCII/UTF-8 box-drawing output for terminals.

**Avoid when:**
- Output is redirected to files/logs and you don't want ANSI escapes to leak.
- You need data-driven tables from CSV/JSON with sorting/paging (add a data layer).
- You target a full interactive TUI (use a terminal-UI framework instead).
- Precise multi-byte alignment across arbitrary locales/platforms is a hard requirement.

## Alternatives

- seleznevae/libfort — C/C++ table library with a plain C API and UTF-8 support; use when you want C compatibility or to avoid heavy C++ templates.
- fmtlib/fmt — general text formatting, not a table library; use when you only need manual column alignment without borders, colors, or word-wrap.
- astanin/python-tabulate — the closest analog in Python; use when your stack is Python rather than C++.
- Textualize/rich — Python terminal rendering with styled tables and markup; use when you need rich TUI-grade output in Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-12-20 | Repository created; first releases of the header-only table maker[^2]. |
| 1.5 | (current) | Latest tagged version per project badge; C++11+ support, Markdown/AsciiDoc exporters[^1]. |
| — | 2025-05-14 | Most recent push; activity quiet since[^2]. |

## References

[^1]: p-ranav/tabulate README (Quick Start, Formatting Options, Exporters, UTF-8 Support). https://github.com/p-ranav/tabulate/blob/master/README.md
[^2]: GitHub API repository metadata for p-ranav/tabulate (stars, forks, created_at 2019-12-20, pushed_at 2025-05-14, open issues, MIT license), retrieved 2026-07. https://github.com/p-ranav/tabulate

## Tags

cpp, cpp11, single-header, header-only, terminal, table, cli, text-formatting, ansi-colors, pretty-print
