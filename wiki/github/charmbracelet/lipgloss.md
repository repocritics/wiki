# charmbracelet/lipgloss

> A declarative, CSS-like styling API for terminal output — the layout layer of Charm's TUI stack.

[GitHub repo](https://github.com/charmbracelet/lipgloss) ·
[License: MIT](https://github.com/charmbracelet/lipgloss/blob/main/LICENSE)

## Overview

Lip Gloss is a Go library for styling and laying out text in the terminal. It
exposes a chainable, immutable `Style` value type whose vocabulary — foreground
and background colors, bold/italic/underline, padding, margins, borders,
alignment, width/height — deliberately mirrors CSS[^1]. You build a `Style`,
call `Render(string)`, and get back a string containing the appropriate ANSI
escape sequences. It renders strings; it does not manage a screen, an event
loop, or input. That job belongs to Bubble Tea, and Lip Gloss is the piece most
Charm TUIs use to draw their views.

The library is part of the Charm ecosystem (Bubble Tea, Bubbles, Glamour,
Huh, Gum) and is most valuable when treated as one layer of that stack rather
than in isolation. Its defining design choice is that a `Style` is a pure value:
assignment copies it, methods return new styles, and nothing mutates in place.
This makes styles trivially composable and safe to share, at the cost of
allocating new strings on every render — a tradeoff that matters inside a tight
Bubble Tea update/view loop.

The other defining tension is color. Terminals vary from truecolor down to
16-color and no-color-at-all, and Lip Gloss's job is to downsample gracefully to
whatever the output supports. In v1 this detection was a process-global set via
`termenv`; v2 (module path `charm.land/lipgloss/v2`) reworked color handling to
be explicit and to integrate with Bubble Tea v2's renderer[^2], which is the
single largest source of migration friction.

## Getting Started

```bash
go get charm.land/lipgloss/v2
```

```go
package main

import "charm.land/lipgloss/v2"

func main() {
	style := lipgloss.NewStyle().
		Bold(true).
		Foreground(lipgloss.Color("#FAFAFA")).
		Background(lipgloss.Color("#7D56F4")).
		Padding(1, 4).
		Width(22)

	lipgloss.Println(style.Render("Hello, kitty"))
}
```

`lipgloss.Println` / `lipgloss.Sprint` are the standalone entry points that
detect the terminal's color profile and downsample accordingly. Inside Bubble
Tea, rendering goes through the program's renderer and this is handled for you.

## Architecture / How It Works

A `Style` is a struct of set/unset rule fields plus their values. Setters like
`.Bold(true)` return a copy with one field marked set; `.UnsetBold()` clears it.
`Inherit(other)` copies only rules the receiver has not already set. Because the
type is a plain value, `b := a` is a genuine copy and styles are safe to pass
around without aliasing bugs.

`Render` walks the set rules and emits SGR escape sequences around the text,
then applies block-level rules (padding, then borders, then margins) by
manipulating lines as strings. Width and height measurement is grapheme- and
East-Asian-width aware (via the `x/ansi` and rune-width machinery), so
`lipgloss.Width` counts display cells rather than bytes or runes — necessary for
CJK text and emoji, and a frequent source of surprise for anyone assuming
`len(s)`.

Color is the layer that changed most between versions. A color profile
(truecolor / 256 / 16 / ASCII) is chosen from the environment (`TERM`,
`COLORTERM`, `NO_COLOR`, and whether output is a TTY). Colors specified as hex
or ANSI indices are downsampled to the nearest value the active profile
supports. v1 stored this profile globally through `termenv`; v2 makes it
explicit and delegates detection to Bubble Tea v2 / the `colorprofile` package,
which removes a class of bugs around concurrent renderers and tests but breaks
v1 code that relied on the global.

Layout helpers sit on top of the core: `JoinHorizontal` / `JoinVertical` glue
blocks along an axis, `Place` / `PlaceHorizontal` / `PlaceVertical` position a
block within whitespace, and v2 adds a cell-based compositor (`NewLayer`,
`Compose`) for overlapping, z-ordered layers. Three sub-packages —
`table`, `list`, and `tree` — are separate renderers that produce styled strings
using the same `Style` type, each with a `StyleFunc` hook for per-cell/-item
styling.

## Production Notes

**Rendering allocates.** Every `Render` builds new strings, and every style
method returns a copy. In a Bubble Tea `View()` that runs on each frame, styling
the same static content repeatedly is wasted allocation. The standard mitigation
is to construct styles once (package-level vars) and reuse them, and to avoid
re-rendering unchanging sub-views.

**Width math is the top footgun.** `Width`/`Height`/`Size` measure display
cells, not runes. Emoji, combining marks, and CJK text can be one rune but two
cells; ANSI escapes count as zero. Manual truncation with `s[:n]` will corrupt
styled or wide text — use Lip Gloss's own width helpers, `MaxWidth`, or
`Inline(true)` when you need hard limits.

**Color detection depends on environment, not just the terminal.** In CI, when
piped, or when logging, output is typically not a TTY and colors are stripped or
downsampled to ASCII. Honor `NO_COLOR`; do not assume truecolor. If you set
colors as hex and see them flattened, the profile — not the code — is usually
the cause.

**The v1 → v2 migration is a real port, not a version bump.** The module path
changes to `charm.land/lipgloss/v2`, global color-profile handling is removed in
favor of explicit/Bubble-Tea-mediated detection, and several APIs (color
utilities, compositor, blending) are new or reshaped. Charm ships an
`UPGRADE_GUIDE_V2.md`[^3]; budget time to move color setup off the old global
and to re-verify rendering in your target terminals. v1 remains importable at
its original path for projects not ready to move.

**Standalone vs. Bubble Tea.** If you use Lip Gloss outside Bubble Tea, route
output through `lipgloss.Println`/`Sprint` (or configure a writer) so profile
detection happens. Rendering styled strings straight to `fmt.Println` bypasses
downsampling and can emit truecolor escapes to terminals that mangle them.

## When to Use / When Not

**Use when:**
- You are building a Bubble Tea TUI and need to style and lay out its views.
- You want CSS-like, composable styling for terminal output without hand-writing
  ANSI escapes.
- You need tables, lists, or trees rendered to styled text.
- You need reliable display-width measurement for wide/emoji/CJK content.

**Avoid when:**
- You need cursor movement, alt-screen control, or input handling — that is
  Bubble Tea (or a lower-level library), not Lip Gloss.
- Output is machine-consumed (logs, pipes) where escape sequences are noise;
  prefer plain text or a structured logger.
- You are in a hot path where per-frame string allocation is unacceptable and
  cannot cache rendered output.

## Alternatives

- muesli/termenv — the lower-level color/profile primitive Lip Gloss built on;
  use it when you want profile handling without the styling/layout DSL.
- gdamore/tcell — use when you need a full cell-grid screen abstraction with
  input, not string styling.
- rivo/tview — use when you want ready-made widgets on top of tcell rather than
  composing your own views.
- fatih/color — use for simple colored log/CLI output where borders, padding,
  and layout are unnecessary.
- jwalton/gchalk — a chalk-style Go alternative when you only need inline text
  coloring.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-03 | First public release; declarative `Style` type, CSS-like API[^1]. |
| v0.x | 2022–2024 | `table`, `list`, and `tree` sub-packages added over the v0 series. |
| v2 | 2026 | Module path `charm.land/lipgloss/v2`; explicit color profiles, compositor/layers, color-blending utilities, Bubble Tea v2 integration[^2][^3]. |

## References

[^1]: Lip Gloss README — declarative, CSS-inspired styling API. https://github.com/charmbracelet/lipgloss
[^2]: Charm — Bubble Tea v2 and the reworked rendering/color-profile stack. https://github.com/charmbracelet/bubbletea
[^3]: Lip Gloss v2 upgrade guide. https://github.com/charmbracelet/lipgloss/blob/main/UPGRADE_GUIDE_V2.md

## Tags

go, golang, tui, terminal, cli, styling, layout, ansi, charm, bubbletea, text-rendering
