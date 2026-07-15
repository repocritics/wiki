# urwid/urwid

> A widget-and-canvas console UI library for Python — the long-standing, pre-Textual way to build full-screen terminal applications.

[GitHub repo](https://github.com/urwid/urwid) ·
[Official website](https://urwid.org) ·
[License: LGPL-2.1](https://github.com/urwid/urwid/blob/master/COPYING)

## Overview

Urwid is a terminal user interface (TUI) library for Python, created by Ian Ward
and first released in the mid-2000s, predating its 2010 migration to GitHub[^1].
It gives you a widget tree, a rendering model based on canvases, and a main loop
that owns the terminal — the same conceptual stack as a GUI toolkit, projected
onto a character grid. If you have used `dialog`, `htop`, or a curses program and
wanted to build one in Python without hand-managing cursor positions, urwid is
the library that has covered that need for roughly two decades.

It targets Linux, macOS, Cygwin and other unix-likes, with partial Windows 10+
support added recently (no `Terminal` widget, sockets only, UTF-8 only)[^2]. It
supports UTF-8 and CJK encodings, and 88 / 256 / 24-bit color where the terminal
allows.

The defining tension is age. Urwid's API predates async/await, type hints, and
the modern "reactive UI" style, and it shows: the widget sizing model (flow vs.
box vs. fixed) is powerful but famously unintuitive, and much of the ecosystem's
newer energy has moved to Textual. Urwid's counter-argument is stability — it is
small, dependency-free at its core, event-loop-agnostic, and still actively
maintained, with recent releases adding type annotations and Windows support[^2].

## Getting Started

```bash
pip install urwid
# extras, e.g. ZeroMQ event loop + serial LCD display:
pip install "urwid[serial,zmq]"
```

```python
import urwid

def on_key(key):
    if key in ("q", "Q"):
        raise urwid.ExitMainLoop()

txt = urwid.Text(("banner", "Hello, urwid.  Press q to quit."), align="center")
fill = urwid.Filler(txt, valign="middle")
palette = [("banner", "black", "light gray")]

loop = urwid.MainLoop(fill, palette, unhandled_input=on_key)
loop.run()
```

`Text` is a flow widget; `Filler` is the box adapter that gives it vertical
room; `MainLoop` takes over the terminal, runs the event loop, and only returns
when `ExitMainLoop` is raised.

## Architecture / How It Works

Urwid separates four layers, and understanding the seams is most of the learning
curve:

1. **Widgets** — the tree you build. Leaf widgets (`Text`, `Edit`, `Button`,
   `CheckBox`, `RadioButton`, `SelectableIcon`) and container widgets (`Pile`,
   `Columns`, `Frame`, `Overlay`, `GridFlow`, `LineBox`, `Padding`, `Filler`,
   `ListBox`). Widgets don't hold pixel coordinates; they render into a canvas
   for a size that a parent supplies.
2. **Sizing model** — every widget is a *flow* widget (you give it a width, it
   computes its height), a *box* widget (you give it width and height), or a
   *fixed* widget (it reports its own size). `ListBox` is a box widget; `Text`
   is a flow widget; putting one where the other is expected is the single most
   common source of `WidgetError`/`assert` failures for newcomers. Adapters
   (`Filler`, `BoxAdapter`) exist specifically to convert between them.
3. **Canvas** — rendering produces a `Canvas`: a grid of text plus per-cell
   display attributes. Canvases are cached (`CanvasCache`) and composited, so
   only changed regions are redrawn. This is what makes redraws cheap without a
   diffing framework.
4. **Display + MainLoop** — a `Screen` (`raw_display`, `curses_display`, plus
   experimental `web_display` / `lcd_display`) owns the terminal I/O, and
   `MainLoop` drives input, timers, and redraws. Crucially the loop is
   pluggable: `SelectEventLoop` by default, or Twisted / GLib / Tornado /
   asyncio / trio / ZeroMQ, which is how urwid embeds inside other async
   applications[^2].

Cross-widget communication uses a signals system (`urwid.connect_signal`,
`emit_signal`) rather than callbacks threaded through constructors, and colors
are declared once as a *palette* and referenced by name inside markup tuples
like `("banner", "text")`. Scrolling lists are driven by a `ListWalker`
(`SimpleListWalker`, `SimpleFocusListWalker`, or a custom one) — the walker, not
the `ListBox`, owns the content model, which is what lets `ListBox` scroll
arbitrarily large or lazily-generated content.

## Production Notes

- **The sizing model is the footgun.** Most real-world urwid bugs are a flow
  widget handed a box context or vice versa. Learn `('weight', n)`,
  `('given', n)`, `('pack', None)` for `Columns`/`Pile` early; they are not
  optional polish.
- **Wide/combining characters.** CJK and combining Unicode are supported but
  width calculation depends on the terminal agreeing with urwid's tables.
  Mismatches cause off-by-one rendering and cursor drift; set the encoding
  explicitly with `urwid.set_encoding("utf8")` and test on your real terminals.
- **Windows is partial, not first-class.** No `Terminal` widget, only sockets as
  file descriptors, no `ZMQEventLoop`, UTF-8 only, and mouse input is unreliable
  on fast actions[^2]. Treat Windows as best-effort.
- **`raw_display` vs `curses_display`.** `raw_display` is the default and more
  featureful (true color, better mouse); `curses_display` exists for
  environments where raw escape handling is a problem. They are not perfectly
  interchangeable — color depth and input handling differ.
- **Redraw discipline.** Urwid does not auto-redraw on state change; you mutate
  widget attributes and the loop repaints on its schedule. Long synchronous work
  in a callback freezes the UI — offload to the event loop's timers/executor.
- **Terminal restoration.** An uncaught exception can leave the terminal in raw
  mode. `MainLoop.run()` restores on clean exit and on `ExitMainLoop`, but wrap
  risky code so a crash doesn't leave the user with a broken shell.

## When to Use / When Not

**Use when:**
- You need a full-screen TUI in Python and want a mature, small, dependency-free
  core.
- You must embed the UI inside an existing Twisted / asyncio / GLib / trio
  application — urwid's event-loop pluggability is its strongest differentiator.
- You want fine, low-level control over the canvas and rendering.

**Avoid when:**
- You're starting fresh and want modern DX (CSS-like styling, async widgets,
  hot-reload, web export) — Textual is the more productive choice today.
- You only need a rich prompt / line editor rather than a full-screen app —
  prompt-toolkit is a better fit.
- You need dependable Windows support or a large library of ready-made,
  styled components out of the box.

## Alternatives

- Textualize/textual — modern async Python TUI with CSS-like styling and a web
  target; use it when starting a new project and DX matters more than legacy.
- prompt-toolkit/python-prompt-toolkit — use instead when you need line editing,
  autocompletion, and REPL-style prompts rather than a full-screen widget tree.
- charmbracelet/bubbletea — use when you're not tied to Python and want the
  Elm-architecture model in Go.
- rivo/tview — use for a Go, widget-oriented TUI with a component set close to
  urwid's spirit.
- gizak/termui — use when the app is dashboard/charts-first rather than
  interactive-form-first.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.x | mid-2000s | Early releases by Ian Ward; core widget/canvas model established[^1]. |
| 1.0.0 | 2012 | First 1.x; API consolidation. |
| 2.0.0 | 2018 | Dropped Python 2 support. |
| 2.1.x | 2020 | Packaging and event-loop refinements. |
| 2.6.x | 2024 | Type annotations, partial Windows 10+ support, Python 3.13/3.14 in CI[^2]. |

## References

[^1]: urwid project home and history. https://urwid.org
[^2]: urwid README, "Windows support notes" / "Supported Python versions" / event loops. https://github.com/urwid/urwid/blob/master/README.rst

## Tags

python, tui, terminal, console-ui, ncurses, widget-toolkit, cli, text-user-interface, event-loop, cross-platform
