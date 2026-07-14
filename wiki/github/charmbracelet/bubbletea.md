# charmbracelet/bubbletea

> A Go framework for building terminal UIs on The Elm Architecture — model, update, view, and message-driven I/O.

[GitHub repo](https://github.com/charmbracelet/bubbletea) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/charmbracelet/bubbletea) ·
[License: MIT](https://github.com/charmbracelet/bubbletea/blob/main/LICENSE)

## Overview

Bubble Tea is a Go framework for terminal user interfaces, built by Charm (charmbracelet) and first released in 2020[^1]. It ports [The Elm Architecture][tea] — a strict `Model` / `Update` / `View` loop driven by immutable messages — to Go. Application state lives in a single model; events arrive as `tea.Msg` values; `Update` returns a new model plus optional commands; `View` renders the current state to a string. The runtime owns the terminal, the event loop, and redraw scheduling, so application code never touches raw escape sequences or the draw cycle directly.

The framework anchors a broad ecosystem. [charmbracelet/bubbles][bubbles] supplies stock components (text input, list, table, viewport, spinner, progress), [charmbracelet/lipgloss][lipgloss] handles styling and layout, and Glamour, Huh, Wish, and Gum build on the same primitives. Most non-trivial Bubble Tea apps pull in Bubbles and Lip Gloss rather than rendering by hand. It is one of the most widely used TUI frameworks in Go, with tens of thousands of dependent projects including tools shipped by Microsoft Azure, AWS, NVIDIA, and Cockroach Labs[^2].

The defining tension is the Elm model itself. The message-passing discipline makes state transitions explicit and testable, but it forces all I/O through `Cmd` values and all concurrency through messages — a real adjustment for developers used to imperative UI code. The reward is that redraw logic disappears; the cost is that "just do this side effect here" is never the answer.

## Getting Started

```bash
go get github.com/charmbracelet/bubbletea
```

A minimal counter. `Update` mutates a value-receiver copy and returns it; `View` returns the frame as a string:

```go
package main

import (
	"fmt"
	"os"

	tea "github.com/charmbracelet/bubbletea"
)

type model int

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	if key, ok := msg.(tea.KeyMsg); ok {
		switch key.String() {
		case "up", "k":
			m++
		case "down", "j":
			m--
		case "q", "ctrl+c":
			return m, tea.Quit // special command: exit the runtime
		}
	}
	return m, nil
}

func (m model) View() string {
	return fmt.Sprintf("count: %d\n(up/down to change, q to quit)\n", int(m))
}

func main() {
	if _, err := tea.NewProgram(model(0)).Run(); err != nil {
		fmt.Println("error:", err)
		os.Exit(1)
	}
}
```

Side effects go through commands: a `tea.Cmd` is a `func() tea.Msg` run by the runtime off the render path, and its returned message re-enters `Update`.

## Architecture / How It Works

A `Program` runs a single event loop over three model methods:

1. **`Init() tea.Cmd`** — returns an initial command (or `nil`) to kick off startup I/O.
2. **`Update(tea.Msg) (tea.Model, tea.Cmd)`** — the only place state changes. It receives one message at a time, returns the next model and an optional command.
3. **`View() string`** — pure function of the current model, returns the full frame.

**Messages and commands** are the concurrency model. There are no callbacks and no shared mutable UI state. Async work (an HTTP call, a timer, a file read) is expressed as a `Cmd`; the runtime executes it in a goroutine and feeds the resulting `Msg` back into `Update` serially. Because `Update` is called one message at a time, application state is effectively single-threaded and free of locks. External goroutines push events in with `Program.Send`.

**The renderer** diffs frames and writes only what changed. The classic ("standard") renderer is line-based and throttles to roughly 60 FPS to avoid flooding the terminal; batching means a burst of messages coalesces into one repaint. Full-window apps opt into the alternate screen buffer (`tea.WithAltScreen`); inline apps render in place. Mouse tracking, bracketed paste, focus reporting, and window-resize events (`tea.WindowSizeMsg`) are surfaced as ordinary messages.

**Composition** is manual. Bubble Tea has no widget tree or parent/child wiring; a parent model embeds child models as struct fields and forwards messages to their `Update` methods, threading the returned child model and command back up. This keeps the core tiny but means layout and event routing across many components is code you write, typically leaning on Lip Gloss for the visual assembly.

## Production Notes

**Never block in `Update`.** A slow call there freezes the entire UI, because the event loop cannot process the next message — including keypresses — until `Update` returns. All I/O and sleeps belong in commands. This is the single most common beginner mistake.

**Value vs pointer receivers.** The canonical style uses value receivers and returns the modified model. If a model is large or holds maps/slices you mutate in place, be deliberate: returning `m` copies the struct each update, and in-place mutation of reference fields can produce aliasing surprises. Pointer receivers work but break the "return the new model" idiom that examples assume.

**External events use `Program.Send`.** To feed a goroutine's output into the loop, capture the `*tea.Program` and call `p.Send(msg)`. A frequent bug is starting goroutines inside `Update` and trying to update the model from them directly — state must round-trip through a message.

**Logging can't go to stdout**, which the TUI owns. Use `tea.LogToFile` and `tail -f` the file, and run Delve headless (`dlv debug --headless --listen=...`) since the debugger and the app both want the terminal.

**Testing.** The `teatest` package (`x/exp/teatest`) drives a program with scripted input and asserts on output frames; because models are plain values with a pure `Update`, unit-testing transitions without the runtime is also straightforward.

**The v1 → v2 migration is the real upgrade pain.** v2 is a substantial API break: it splits `KeyMsg` into press/release messages, changes the `View` return type, and moves the canonical import path to `charm.land/bubbletea/v2`[^3]. Most published tutorials, blog posts, and Stack Overflow answers describe v1, and Bubbles/Lip Gloss versions must be matched to the Bubble Tea major you target. Pin versions deliberately and consult the upstream upgrade guide before mixing v1 and v2 code.

## When to Use / When Not

**Use when:**
- You want an interactive, stateful TUI (dashboards, wizards, pickers, full-screen apps) and value explicit, testable state transitions.
- You're already in the Charm ecosystem and want Bubbles components with Lip Gloss styling.
- You need fine control over keyboard, mouse, paste, and resize handling.

**Avoid when:**
- You just need colored output, a spinner, or a prompt — reach for Lip Gloss, `huh`, `promptui`, or `pterm` directly; the Elm loop is overkill.
- Your team won't invest in the message-passing model and wants imperative widgets with built-in event handlers (tview fits better).
- You're not in Go, or you need a mature high-level widget library out of the box.

## Alternatives

- rivo/tview — batteries-included Go widget library with built-in layouts and event handlers; use it when you want ready-made widgets and an imperative model instead of Elm.
- gizak/termui — Go dashboard/charting toolkit; use it when the app is primarily gauges, charts, and grids.
- awesome-gocui/gocui — minimal Go library for managing overlapping views; use it when you want a thin layer over raw terminal control.
- ratatui/ratatui — the leading Rust TUI library (immediate-mode); use it when you're writing Rust and want performance plus a large widget set.
- textualize/textual — Python TUI framework with CSS-like styling and async; use it when Python is the host language and you want web-inspired layout.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2020-01 | First public release by Charm[^1]. |
| v0.x series | 2020–2024 | Long-lived pre-1.0 line; Bubbles/Lip Gloss ecosystem matured alongside. |
| v1.0.0 | 2024 | First stable major; `github.com/charmbracelet/bubbletea` import path. |
| v2 | 2025 | Major API break: split key messages, new `View` type, `charm.land/bubbletea/v2` path[^3]. |

## References

[^1]: charmbracelet/bubbletea repository and release history. https://github.com/charmbracelet/bubbletea/releases
[^2]: "Bubble Tea in the Wild" and dependents graph, project README. https://github.com/charmbracelet/bubbletea/network/dependents
[^3]: v2 upgrade guide, project repository. https://github.com/charmbracelet/bubbletea/blob/main/UPGRADE_GUIDE_V2.md

[tea]: https://guide.elm-lang.org/architecture/
[bubbles]: https://github.com/charmbracelet/bubbles
[lipgloss]: https://github.com/charmbracelet/lipgloss

## Tags

go, golang, tui, terminal, cli, elm-architecture, framework, functional, charm, ui
