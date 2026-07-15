# jedib0t/go-pretty

> Console output formatting for Go — tables, lists, progress bars, and ANSI-aware text, with customization as the organizing principle.

[GitHub repo](https://github.com/jedib0t/go-pretty) ·
[Go Reference](https://pkg.go.dev/github.com/jedib0t/go-pretty/v6) ·
[License: MIT](https://github.com/jedib0t/go-pretty/blob/main/LICENSE)

## Overview

go-pretty is a Go library for rendering structured console output: ASCII/Unicode
tables, nested lists, live progress bars, and low-level ANSI text utilities. It has
been developed since 2018[^1] and is one of the two libraries most Go developers
reach for when they need a formatted table (the other being olekukonko/tablewriter).
Its distinguishing trait is breadth of configuration: styles, per-column alignment
and width constraints, auto-merging cells, sorting, paging, and five output encoders
(ASCII, HTML, Markdown, CSV, TSV) are all first-class table features rather than
add-ons.

The library is organized as four largely independent packages — `table`, `list`,
`progress`, `text` — under one module. `text` is the shared foundation: it handles
alignment, color/formatting escape codes, wrapping, and ANSI-aware string length,
and the other three packages build on it. You import only the sub-packages you use,
so pulling in `table` does not drag in `progress`.

The defining tension is expressiveness versus simplicity. go-pretty exposes a large
surface (the `table.Writer` interface has dozens of methods) and renders the whole
output into memory as a string. That makes it excellent for one-shot formatted
reports in CLIs and excellent to customize, but it is not a streaming or
constant-memory renderer, and the configuration API is verbose compared to
minimalist alternatives.

## Getting Started

```bash
go get github.com/jedib0t/go-pretty/v6
```

The module is at major version **v6**; the `/v6` suffix in the import path is
mandatory and is the single most common setup mistake[^2].

```go
package main

import (
	"os"

	"github.com/jedib0t/go-pretty/v6/table"
)

func main() {
	t := table.NewWriter()
	t.SetOutputMirror(os.Stdout)
	t.AppendHeader(table.Row{"#", "First Name", "Last Name", "Salary"})
	t.AppendRows([]table.Row{
		{1, "Arya", "Stark", 3000},
		{20, "Jon", "Snow", 2000, "You know nothing, Jon Snow!"},
	})
	t.AppendFooter(table.Row{"", "", "Total", 5000})
	t.SetStyle(table.StyleLight)
	t.Render() // also RenderHTML / RenderMarkdown / RenderCSV / RenderTSV
}
```

## Architecture / How It Works

The `table` package centers on the `table.Writer` interface. You append `Row`
values (`[]interface{}`), configure via `SetStyle`, `SetColumnConfigs`,
`SortBy`, `SetPageSize`, and similar setters, then call one of the `Render*`
methods. Rendering is a single synchronous pass that computes column widths,
applies alignment and merging, and assembles the full output string. Styles are
plain structs (`table.Style` with `Box`, `Color`, `Format`, `Options`
sub-structs), so a custom look is a struct literal you can copy and mutate rather
than a builder chain.

The hard problem underneath all of this is measuring *visible* width. Terminal
output mixes ANSI escape sequences (which occupy zero columns) with East-Asian
wide and combining Unicode characters (which occupy two or zero). The `text`
package provides ANSI-aware length and manipulation, and the library relies on
runewidth-style width calculation so that CJK characters, box-drawing glyphs, and
colored cells still align. This is why alignment "just works" in colored tables
and why width bugs, when they occur, trace back to width measurement of unusual
glyphs.

The `progress` package works differently: it runs a rendering goroutine that
repaints the tracker block on an interval using ANSI cursor-movement escapes.
You create a `progress.Writer`, register `*progress.Tracker` values, call
`Render()` (typically in a goroutine), and increment trackers from your worker
code. It supports determinate and indeterminate modes, ETA and speed estimation,
and per-tracker styling.

`list` renders hierarchical bullet trees (`AppendItem` / `Indent` / `UnIndent`)
to ASCII, HTML, or Markdown, reusing the same style philosophy.

## Production Notes

- **The `/v6` import path is a hard requirement.** Because Go modules encode the
  major version in the path, `go get github.com/jedib0t/go-pretty` (without
  `/v6`) resolves to the ancient v1 API and produces confusing "undefined"
  errors. Always import `.../go-pretty/v6/...`.
- **Rendering is buffered, not streamed.** The entire table is built as one string
  in memory before output. For very large tables (tens of thousands of rows) this
  costs proportional memory and there is no incremental/row-at-a-time flush. Use
  `SetPageSize` to chunk output, or a CSV encoder + `encoding/csv` for bulk data.
- **Progress bars need a real TTY to look right.** The `progress` renderer uses
  cursor-movement escapes; when stdout is redirected to a file or a CI log, the
  repaint sequences produce noisy or garbled output. Detect non-interactive
  terminals and disable or simplify the tracker in those environments.
- **Width miscalculation on exotic glyphs.** Alignment depends on rune-width
  tables; multi-codepoint emoji, ZWJ sequences, and some ambiguous-width East-Asian
  characters can render one column off. This is a general terminal-width problem,
  not unique to go-pretty, but it surfaces here as misaligned borders.
- **Colors depend on terminal capability and are auto-suppressed** when output is
  not a terminal in typical setups; verify behavior in your target CI/log sink
  rather than assuming ANSI passes through.
- **Verbose configuration.** Non-trivial layouts (per-column width limits,
  transformers, merges, sort keys) require several setter calls. This is by design
  but makes the call sites longer than minimalist table libraries.

## When to Use / When Not

**Use when:**
- You are building a CLI or admin tool that prints formatted tables, trees, or
  progress and want colors, alignment, and multiple output formats without wiring
  them yourself.
- You need one library to emit the same tabular data as ASCII, Markdown, HTML, and
  CSV/TSV.
- You want fine control over styling (borders, colors, merges, per-column rules).

**Avoid when:**
- You need a full interactive TUI (menus, viewports, event loop) — reach for a
  framework like charmbracelet/bubbletea with lipgloss instead.
- You are streaming millions of rows and cannot hold the rendered output in memory.
- You want the smallest possible dependency and API for a plain table — a minimal
  library will be terser.

## Alternatives

- olekukonko/tablewriter — the other widely used Go table library; use it when you
  want a smaller, table-only dependency and don't need lists/progress/encoders.
- charmbracelet/lipgloss — style and layout primitives (including tables) for
  terminal UIs; use it when you are building a Bubble Tea TUI and want styling to
  match.
- charmbracelet/bubbletea — full TUI framework; use it when you need interactivity,
  not one-shot rendering.
- schollz/progressbar — dedicated single progress bar; use it when a progress bar
  is the only thing you need.
- vbauerster/mpb — multi-bar progress rendering; use it when concurrent progress
  bars are your primary requirement.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-04-30 | Repository created; table-writer origins[^1]. |
| v6.x | current | Active major line; module path `.../go-pretty/v6`. Four packages: table, list, progress, text[^2]. |

The project follows Go's major-version-in-path convention, so each major release
(through v6) changed the import suffix; v6 is the current stable line and where all
development happens.

## References

[^1]: jedib0t/go-pretty repository, created 2018-04-30. https://github.com/jedib0t/go-pretty
[^2]: Go Reference and README, module `github.com/jedib0t/go-pretty/v6`. https://pkg.go.dev/github.com/jedib0t/go-pretty/v6

## Tags

go, golang, cli, terminal, table, console-output, progress-bar, ansi, text-formatting, tui, library
