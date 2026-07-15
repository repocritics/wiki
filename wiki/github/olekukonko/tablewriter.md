# olekukonko/tablewriter

> A Go library for rendering text tables — ASCII, Unicode, Markdown, HTML, and colorized terminal output.

[GitHub repo](https://github.com/olekukonko/tablewriter) ·
[License: MIT](https://github.com/olekukonko/tablewriter/blob/master/LICENSE)

## Overview

`tablewriter` renders tabular data as text: box-drawing ASCII/Unicode tables for
CLIs, Markdown tables for documentation, HTML, and ANSI-colorized terminal
output. It has been one of the most-depended-on utility packages in the Go
ecosystem since the mid-2010s — pulled in transitively by countless CLI tools,
Kubernetes-adjacent tooling, and internal dashboards — largely on the strength
of a tiny, stable API around `NewWriter` / `Append` / `Render`[^1].

The defining fact of this project in 2026 is the **v0.0.5 → v1.x rewrite**. For
years the canonical version was `v0.0.5` (2021), a single-package library with a
mutation-heavy builder API. The v1 line is a near-total redesign: a
renderer/config architecture, functional options (`WithRenderer`, `WithConfig`),
generics, streaming, and pluggable output backends. The two are not
source-compatible. Compounding this, `v1.0.0` shipped with missing
functionality and the maintainer explicitly warns against it[^2]; the usable v1
line starts at `v1.0.1`+ and stabilized through the `v1.1.x` series.

The practical consequence: the ecosystem is split. A large installed base remains
pinned to `v0.0.5` (it still works and is effectively frozen), while new code and
the README's examples target `@latest`. Anyone reading Stack Overflow answers or
old blog posts is likely reading v0.0.5 API that will not compile against v1.

## Getting Started

```bash
# Legacy line — stable, frozen, what most existing code uses
go get github.com/olekukonko/tablewriter@v0.0.5

# Current line — renderers, generics, streaming
go get github.com/olekukonko/tablewriter@latest
```

Minimal v1 example:

```go
package main

import (
	"os"

	"github.com/olekukonko/tablewriter"
)

func main() {
	table := tablewriter.NewTable(os.Stdout)
	table.Header("Name", "Age", "City")
	table.Bulk([][]any{
		{"Alice", 25, "New York"},
		{"Bob", 30, "Boston"},
	})
	table.Render()
}
```

The default renderer emits Unicode box-drawing characters. Note `NewTable` (v1)
vs `NewWriter` (both lines); the v0.0.5 idiom was `NewWriter(w)` followed by
`SetHeader`, `Append`, and `Render`.

## Architecture / How It Works

The v1 design separates *what* a table contains from *how* it is drawn:

- **Renderer** — the output backend. Built-ins: `Blueprint` (ASCII/Unicode
  box-drawing, the default), `Markdown`, `HTML`, `Colorized` (ANSI, backed by
  `fatih/color`), `Ocean` (streaming ASCII), and an SVG renderer. Renderers
  implement a common interface, so custom output formats are possible.
- **Config** — table behavior: per-section (`Header`/`Row`/`Footer`) formatting,
  padding, alignment, width constraints, auto-wrapping, and callbacks. Applied
  via `WithConfig` or the `Configure(func(*Config))` closure.
- **Rendition** — visual styling passed to a renderer: `Borders`, `Lines`,
  `Separators`, and `Symbols`. `Symbols` is the character set for corners and
  junctions; `NewSymbolCustom` lets you swap in arbitrary glyphs.

Data enters through `Header`, `Append` (one row), and `Bulk` (many). Values are
stringified via `Format()`, then `String()`, then `fmt`; the `sql.Null*` types
and `fmt.Stringer` are handled natively. Cell **merging** is a first-class
feature: horizontal, vertical, `MergeBoth`, and `MergeHierarchical` (repeated
leading columns collapse into a tree, useful for org charts and grouped data).

**Streaming** (`Ocean` renderer + `WithStreaming`) renders row-by-row as data
arrives rather than buffering the whole table. Because column widths must be
fixed before the first row prints, streaming mode truncates rather than
reflows — a fundamentally different width model from the batch renderers, which
measure all rows before drawing.

## Production Notes

- **Pin your version deliberately.** `v0.0.5` and `v1.x` are different libraries.
  A blind `@latest` bump on code written against v0.0.5 will not compile —
  `SetHeader`/`SetBorder`/`SetAlignment` and friends are gone, replaced by the
  config/renderer model. Budget real migration time, not a version bump.
- **Skip `v1.0.0`.** It is documented as incomplete; start at a later `v1.0.x`
  or `v1.1.x`[^2].
- **Width and wrapping are the main footguns.** East Asian wide characters,
  emoji, and combining runes make display-width computation imperfect; expect
  occasional misalignment with mixed-width content. ANSI color codes must be
  length-excluded from width math, which the `Colorized` renderer handles but
  hand-rolled colored strings passed as cell values will not.
- **Not a data grid.** This is a text formatter, not a paginating/sorting table.
  For very large datasets, batch renderers hold all rows in memory to compute
  widths; use streaming mode (with its truncation tradeoff) if that matters.
- **Markdown output is for docs, not untrusted rendering.** Cell contents are
  escaped for the target format, but the library's job ends at producing a
  string; downstream Markdown/HTML handling is your responsibility.
- **Active maintenance.** With ~4.8k stars and commits landing through 2026, the
  project is maintained — but the churn is now concentrated in the v1 line, and
  v0.0.5 receives no changes.

## When to Use / When Not

**Use when:**
- You need pretty terminal tables in a Go CLI with minimal ceremony.
- You want one library that emits ASCII, Markdown, and HTML from the same data.
- You need cell merging, hierarchical grouping, or per-column styling.

**Avoid when:**
- You want a stable, never-changing dependency and are unwilling to track the
  v0.0.5-vs-v1 split — the API surface is in flux relative to its long-frozen past.
- You need rich TUI layout (boxes, flexbox, live updates) — reach for a full TUI
  toolkit instead.
- You render enormous datasets and cannot afford full-table buffering, and the
  streaming renderer's truncation behavior is unacceptable.

## Alternatives

- jedib0t/go-pretty — use when you want tables plus lists and progress bars in
  one styled, actively maintained package with more layout options.
- charmbracelet/lipgloss — use when your CLI already uses the Charm stack and you
  want tables that share its styling model (its `table` subpackage).
- rodaine/table — use when you want the smallest possible dependency for plain
  aligned columns and nothing else.
- aquasecurity/table — use when you want simple bordered, colored tables with a
  compact API and fewer moving parts than v1 tablewriter.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo start) | 2014-02 | Created; long pre-tag history as the de facto Go table lib[^1]. |
| v0.0.1 | 2018-10 | First tagged release. |
| v0.0.5 | 2021-02 | Legacy stable — the version most existing code pins to[^2]. |
| v1.0.0 | 2025-05 | Rewrite: renderer/config architecture. Shipped incomplete — avoid[^2]. |
| v1.1.0 | 2025-09 | v1 line stabilizing: generics, streaming (`Ocean`), merging. |
| v1.1.4 | 2026-03 | Current stable at time of writing[^3]. |

## References

[^1]: olekukonko/tablewriter — repository and README, "TableWriter for Go".
  https://github.com/olekukonko/tablewriter
[^2]: README version guidance: use `v0.0.5` (legacy stable) or `@latest`;
  "Version `v1.0.0` contains missing functionality and should not be used."
  https://github.com/olekukonko/tablewriter/blob/master/README.md
[^3]: Release tags and dates via GitHub API (`/repos/olekukonko/tablewriter/tags`),
  retrieved 2026-07. Latest release `v1.1.4` (2026-03-12).

## Tags

go, golang, cli, ascii-table, table, terminal, text-formatting, markdown, pretty-print, output-formatting
