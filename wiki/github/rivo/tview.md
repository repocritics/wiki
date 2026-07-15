# rivo/tview

> Retained-mode widget library for terminal user interfaces in Go, built on tcell.

[GitHub repo](https://github.com/rivo/tview) ·
[Package docs](https://pkg.go.dev/github.com/rivo/tview) ·
[License: MIT](https://github.com/rivo/tview/blob/master/LICENSE.txt)

## Overview

tview is a Go library of prebuilt, interactive terminal widgets — forms, tables, tree views, lists, text views, editable text areas, flex/grid layouts, modals, and an application wrapper that owns the event loop. It sits one layer above `gdamore/tcell`, the cell-based terminal driver that handles raw input, styling, and screen diffing[^1]. If tcell is "a canvas and an event stream," tview is "the widgets you'd otherwise build on that canvas." It is one of the two dominant Go TUI ecosystems, the other being Charm's Bubble Tea.

The defining architectural choice is that tview is **retained-mode and object-oriented**, not immediate-mode or Elm-style. You construct widget objects, mutate them via chained setters (`SetText`, `SetBorder`, `AddItem`), wire callbacks, and hand a root primitive to `Application.Run()`. This is familiar to anyone who has used a classic GUI toolkit and makes simple apps very short. The tradeoff surfaces under concurrency: the widget tree is shared mutable state driven by a single draw goroutine, so any update from a background goroutine must be marshaled back onto the event loop or you get races and missed redraws. That single rule is the source of most tview bugs in the wild.

The project is maintained primarily by one person (Oliver Kuederle / "rivo") and has been developed continuously since late 2017[^2]. It is widely depended on — the GitHub CLI (`cli/cli`), `derailed/k9s`, `dundee/gdu`, and `containers/podman-tui` are among the well-known consumers[^3].

## Getting Started

```bash
go get github.com/rivo/tview@master
```

```go
package main

import "github.com/rivo/tview"

func main() {
	app := tview.NewApplication()

	form := tview.NewForm().
		AddInputField("Name", "", 20, nil, nil).
		AddDropDown("Role", []string{"Admin", "User"}, 0, nil).
		AddButton("Save", func() { app.Stop() }).
		AddButton("Quit", func() { app.Stop() })

	form.SetBorder(true).SetTitle("Sign up").SetTitleAlign(tview.AlignLeft)

	if err := app.SetRoot(form, true).EnableMouse(true).Run(); err != nil {
		panic(err)
	}
}
```

The README recommends `@master` rather than a pinned tag; see History for why that matters. `SetRoot(p, true)` gives the root primitive the full screen; `EnableMouse` is opt-in.

## Architecture / How It Works

Everything drawable implements the `Primitive` interface — chiefly `Draw(screen tcell.Screen)`, `SetRect`/`GetRect`, and input/focus handlers. `Box` is the base primitive (border, title, padding, background) and every higher widget embeds it, so the widget set is composition over a common rectangle-with-chrome unit.

The runtime is a single event loop inside `Application`. `Run()` reads events from tcell, routes key/mouse events to the focused primitive (bubbling up through `InputHandler`/`MouseHandler`), then redraws. **Drawing happens on one goroutine.** The two escape hatches for touching the UI from elsewhere are `QueueUpdate(func())` and `QueueUpdateDraw(func())` — they push a closure onto the event loop so your mutation runs between draws instead of racing them. `Application.Draw()` requests a redraw from within the loop. Getting this wrong is the canonical footgun: mutate a `TextView` from a network goroutine without queuing and you'll see torn output, nothing at all, or a data race under `-race`.

Layout is done with container primitives, not a constraint solver: `Flex` (one-dimensional, proportional or fixed sizing per child), `Grid` (rows/columns with spanning), and `Pages` (a z-ordered stack, used for modals and view switching). Widgets do not reflow on their own; the container recomputes child rects on each draw from the terminal size tcell reports.

Two dependencies shape behavior. `gdamore/tcell` provides the screen, the event model, and the color/style types you import constantly (`tcell.Color*`, `tcell.KeyEnter`). `rivo/uniseg` provides Unicode grapheme-cluster and width handling, so wide CJK characters and emoji occupy the correct number of cells and don't corrupt column math[^4]. tview re-exports enough of tcell that you rarely import it directly, but the coupling is real: a tcell major change can force a tview change, which the maintainer explicitly names as one of the few things that can break backward compatibility[^2].

## Production Notes

- **QueueUpdateDraw is not optional for concurrent apps.** Any goroutine that updates a widget must go through `QueueUpdate`/`QueueUpdateDraw`. Test under `go test -race` and run with `-race` during development; tview's single-threaded draw model will otherwise hide races until a user hits them.
- **No semantic versioning.** tview publishes commits, not curated releases; consumers pull pseudo-versions (`v0.0.0-<timestamp>-<hash>`) and the README tells you to track `master`. The upside is a strong backward-compatibility ethic — the maintainer states he tries hard not to break existing software[^5]. The downside is you cannot pin to a "1.4.2" and reason about a changelog; upgrades are commit-to-commit and you should read the diff.
- **`Primitive` and other "internal" interfaces are public but unstable.** If you write custom widgets against `Primitive`, know that the maintainer reserves the right to change these interfaces without it counting as a compat break[^5].
- **Terminal capability variance.** Rendering depends on what tcell detects: 256-color vs truecolor, mouse support, and whether the terminal handles the SIXEL or Unicode-block path used by the `Image` widget. Behavior over SSH, inside tmux, or on Windows consoles can differ from a local iTerm/Kitty session. Test on your actual deployment terminal, not just your dev one.
- **Performance is redraw-bound.** A full `Draw()` repaints the visible tree; tcell diffs against the prior frame before writing to the terminal, so cost scales with what actually changed on screen. Large `Table`s use a cell-content callback (`SetContent`) so you don't materialize every cell, but naive full-table rebuilds on every event show up as latency.
- **Single-maintainer bus factor.** Development is real and ongoing, but issue and PR throughput is gated by one person; ~140 open issues is normal steady state, not a sign of abandonment. Budget for carrying local patches if you need a niche fix merged on your timeline.

## When to Use / When Not

**Use when:**
- You want ready-made forms, tables, and tree views without building widgets from scratch, and you think in objects-and-callbacks rather than message-and-update.
- You're building a config-heavy admin/inspector TUI (Kubernetes, databases, cloud resources) — tview's proven sweet spot.
- You need solid Unicode/CJK width handling and mouse support out of the box.

**Avoid when:**
- Your app is highly concurrent and you'd rather the framework enforce safe state updates for you — Bubble Tea's message model removes the manual `QueueUpdate` discipline.
- You need semver-pinned dependencies and a formal changelog for compliance.
- You want composable styling primitives and a component ecosystem (Charm's Lip Gloss / Bubbles) more than a fixed widget set.
- You need low-level control over every cell — drop to tcell directly.

## Alternatives

- charmbracelet/bubbletea — Elm/MVU architecture; the framework owns state and concurrency via messages. Use instead when you want enforced-safe updates and a component ecosystem (Bubbles, Lip Gloss) over a prebuilt widget set.
- gdamore/tcell — the layer tview is built on. Use instead when you want to render cells and handle events yourself with no widget abstraction.
- jroimartin/gocui — minimalist view/keybinding manager. Use instead for simple multi-pane layouts where you don't need rich widgets.
- gizak/termui — dashboard/chart widgets (gauges, sparklines, plots). Use instead for data-visualization dashboards rather than interactive forms.
- pterm/pterm — styled non-interactive terminal output (tables, spinners, progress). Use instead when you want pretty output, not a full-screen interactive app.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Project start | 2017-12 | Repository created; built on tcell from the outset[^2]. |
| uniseg integration | ~2019 | Grapheme-cluster + width handling via rivo/uniseg[^4]. |
| TextArea widget | 2023 | Editable multi-line input added to the widget set. |
| Image widget | ~2024 | Terminal image rendering (Unicode blocks / SIXEL path). |
| Ongoing (`master`) | 2026-03 | Rolling development; no semver releases, pseudo-version pinning[^5]. |

Dates for individual widgets are approximate; tview does not tag semantic releases, so there is no authoritative version-to-date changelog — treat the git history as the source of truth.

## References

[^1]: gdamore/tcell — cell-based terminal driver tview builds on. https://github.com/gdamore/tcell
[^2]: rivo/tview repository (created 2017-12-15). https://github.com/rivo/tview
[^3]: "Projects using tview" — README consumer list (cli/cli, k9s, gdu, podman-tui, and others). https://github.com/rivo/tview#projects-using-tview
[^4]: rivo/uniseg — Unicode text segmentation and width, used by tview. https://github.com/rivo/uniseg
[^5]: tview README, "Backwards-Compatibility" — `@master` recommendation, unstable internal interfaces, compat caveats. https://github.com/rivo/tview#backwards-compatibility

## Tags

go, golang, terminal-ui, tui, widgets, tcell, cli, retained-mode, terminal, ncurses-alternative
