# pterm/pterm

> A Go library for styled terminal output — colors, tables, progress bars, spinners, trees, and simple interactive prompts — built as composable "printer" components.

[GitHub repo](https://github.com/pterm/pterm) ·
[Official website](https://pterm.sh) ·
[License: MIT](https://github.com/pterm/pterm/blob/master/LICENSE)

## Overview

PTerm is a pure-Go output-styling library authored by Marvin Wendt (MarvinJWendt)[^1]. Its scope is "make ordinary CLI output look good and add a few interactive widgets" — colored text, boxes, headers, tables, bullet/tree lists, bar charts, big ASCII text, progress bars, spinners, and prompt widgets (select, multiselect, confirm, text input). It is not a full-screen TUI framework: there is no event loop, no layout engine, and no component tree with focus management. You call a printer, it writes ANSI to stdout, and control returns to your program.

The organizing idea is the **printer**. Everything is a struct (`TextPrinter`, `TablePrinter`, `ProgressbarPrinter`, `AreaPrinter`, …) with a set of package-level defaults (`pterm.Info`, `pterm.DefaultTable`, `pterm.DefaultSpinner`, `pterm.DefaultBarChart`). You configure a copy with chained `WithX()` methods and then call `.Render()`, `.Println()`, or `.Start()`. This gives a consistent, discoverable API across ~25 components at the cost of a large surface of value-copied builder structs. As of this writing the project reports roughly 5,500 stars and 220 forks, is MIT-licensed, and remains actively maintained (last push 2026-07)[^2]. It is cross-platform, including Windows console handling where raw ANSI is not natively honored on older terminals.

The defining tension: PTerm optimizes for *ease and prettiness of linear output*, not for building interactive applications. If your mental model is "printf but nicer," PTerm fits perfectly. If it is "a dashboard the user navigates," you want Bubble Tea instead.

## Getting Started

```sh
go get github.com/pterm/pterm
```

```go
package main

import "github.com/pterm/pterm"

func main() {
	// Prefixed message printers.
	pterm.Info.Println("Starting build")
	pterm.Success.Println("Compiled 42 files")
	pterm.Warning.Println("2 deprecations")

	// A table. WithData takes [][]string; the first row is the header.
	_ = pterm.DefaultTable.WithHasHeader().WithData(pterm.TableData{
		{"Name", "Language", "Stars"},
		{"pterm", "Go", "5500"},
	}).Render()

	// A spinner around a unit of work.
	spinner, _ := pterm.DefaultSpinner.Start("Fetching…")
	// ... do work ...
	spinner.Success("Done")
}
```

The builder methods return a modified **copy**, so `pterm.DefaultTable.WithHasHeader()` does not mutate the shared default — you must chain onto the returned value or assign it.

## Architecture / How It Works

PTerm is a flat package of printer structs plus a handful of subpackages (`putils` for helpers like `LettersFromString`, `pterm/internal`). There are a few core abstractions:

- **`TextPrinter`** — anything that can `Print`/`Println`/`Sprint`. `pterm.Info`, `pterm.Error`, `pterm.DefaultBasicText`, and styled `Style` values all satisfy it, which is why they are interchangeable in helper APIs.
- **`RenderablePrinter`** — components that build a block of text and emit it once (`Table`, `BarChart`, `Tree`, `Box`, `BulletList`, `BigText`). `Render()` prints; `Srender()` returns the string so you can compose or feed it into an `Area`.
- **`LivePrinter`** — components that redraw over time: `Progressbar`, `Spinner`, `Area`. These `Start()` (returning a running instance and an error), expose `Update`/`Increment`, and must be `Stop()`ed. They move the cursor with ANSI escapes and rewrite lines in place.
- **`Area`** — the primitive the other live printers build on. It owns a rectangular region of the terminal and replaces its contents on each `Update`, leaving earlier output untouched. Rendering a `BarChart` to a string with `Srender()` and pushing it into an `Area` each tick is how "live charts" are done.

**Color and styling** go through `pterm.Color` (the 3/4-bit ANSI palette), `pterm.Style` (a slice of attributes), and `pterm.RGB` for TrueColor. On terminals without TrueColor, RGB values are downsampled automatically. Detection of color support, TTY-ness, and Windows console mode is centralized so that most output degrades gracefully in pipes and CI.

**Global state is deliberate and central.** The `Default*` printers and toggles like `pterm.EnableDebugMessages()`, `pterm.DisableColor()`, and `pterm.SetDefaultOutput(w)` are package-level. This is what makes the API terse, and it is also the main coupling story: the library's behavior is governed by mutable process-global variables.

## Production Notes

- **Global mutable defaults are a shared resource.** Because `pterm.Default*`, the active output writer, and color/debug flags are package-level, a library that calls `pterm.DisableColor()` or reassigns `pterm.Info` changes behavior for everyone in the process. Prefer configuring *local copies* (`p := pterm.DefaultTable.WithHasHeader()`) over mutating the globals, and avoid changing PTerm globals from inside importable packages.
- **Interactive prompts need a real TTY.** The `interactive_select`, `interactive_multiselect`, `interactive_confirm`, and `interactive_textinput` printers read raw keyboard input. Under CI, when stdin is not a terminal, or when output is piped, they cannot function normally — gate them behind a TTY check and provide flag-based fallbacks.
- **Live printers own the cursor.** `Spinner`, `Progressbar`, and `Area` write escape sequences to move and clear lines. Interleaving your own `fmt.Println` with a running live printer corrupts the display. Route all output through PTerm (or the area) while a live printer is active, and always `Stop()` it — a panic that skips `Stop()` can leave the terminal in a mangled state.
- **CI / non-TTY output.** In pipelines, spinner animation frames and cursor moves are usually undesirable. Use `pterm.DisableStyling()` / raw output mode, or detect CI and swap live printers for plain lines. Progress bars in particular can flood CI logs with control characters if left in animated mode.
- **Concurrency.** The library is not built around a single render loop, so multiple goroutines each driving their own live printer will fight over the cursor. The `multiple-live-printers` example shows the supported pattern (a coordinating `MultiPrinter`); ad-hoc concurrent live output is a footgun.
- **Not for full-screen apps.** There is no diffing renderer, input router, or window/resize model. Building a navigable UI on top of PTerm means reinventing what Bubble Tea already provides. Reach for it for output and simple prompts, not for a TUI.
- **Version pinning.** PTerm has spent its life in the `v0.x` series, so it does not carry a Go `/v2`+ major-version import path or SemVer stability guarantee. Pin an exact version in `go.mod` and read release notes before upgrading; minor releases have occasionally adjusted printer defaults and signatures.

## When to Use / When Not

**Use when:**
- You want colored, prefixed log-style messages, tables, trees, or boxes with almost no setup.
- You need a progress bar or spinner around a batch job and a nice summary at the end.
- You want a couple of simple interactive prompts (select / confirm / text input) without adopting a TUI framework.
- Cross-platform pretty output (including Windows) matters and you want it to degrade sanely in CI.

**Avoid when:**
- You are building an interactive, navigable full-screen application — use a TUI framework with an event loop.
- You need strict, framework-free control over ANSI, or want zero global state in a reusable library.
- You only need basic colored strings — a smaller styling library is less surface area.
- You require long-term API stability guarantees from a `v1`+ release.

## Alternatives

- charmbracelet/bubbletea — full Elm-architecture TUI framework with an event loop; use it when the user navigates the interface rather than reads linear output.
- charmbracelet/lipgloss — style/layout primitives (borders, alignment, joins) without printers or widgets; use it when you want styling building blocks and to compose your own rendering.
- jedib0t/go-pretty — tables, lists, and progress with a strong table feature set; use it when tables/progress are the whole job and you want deep table control.
- fatih/color — minimal ANSI color for strings; use it when you only need colored text and nothing else.
- rivo/tview — widget-based TUI (forms, flex layouts, grids) on tcell; use it when you need traditional form/panel screens.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2020-09 | Repository created; printer-component design established[^2]. |
| v0.12.x series | 2021– | Long-lived major line; components (table, tree, bar chart, area, interactive prompts, logger) added incrementally. |
| ongoing | 2026-07 | Actively maintained; ~1,538 unit tests reported in-repo, ~5,500 stars[^3]. |

PTerm has not shipped a `v1.0`; it has remained on the `v0` line throughout, treating the `v0.12.x` series as its stable API surface.

## References

[^1]: PTerm author and project — Marvin Wendt. https://pterm.sh
[^2]: Repository metadata (stars, forks, license, created/pushed dates) from the GitHub API for pterm/pterm, retrieved 2026-07. https://github.com/pterm/pterm
[^3]: PTerm README — feature list, component table, and reported unit-test count. https://github.com/pterm/pterm/blob/master/README.md

## Tags

go, golang, cli, terminal, tui, ansi-colors, pretty-print, progressbar, spinner, table, console, library
