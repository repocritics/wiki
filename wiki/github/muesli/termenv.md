# muesli/termenv

> Terminal capability detection and ANSI styling for Go — the layer that answers
> "what can this terminal actually render?" before you write a single escape code.

[GitHub repo](https://github.com/muesli/termenv) ·
[GoDoc](https://pkg.go.dev/github.com/muesli/termenv) ·
License: MIT

## Overview

termenv is a Go library by Christian Muehlhaeuser (muesli), started in December
2019, that probes the terminal a program is running in — color depth, light/dark
background, feature support — and exposes styling primitives (colors, bold,
underline, cursor movement, alt-screen, mouse modes, clipboard, notifications)
that degrade gracefully to what the terminal can display. Colors are classified
into four profiles (Ascii, ANSI 16, ANSI256, TrueColor) and any input color is
converted down to the nearest available one, using HSLuv color space distance
since v0.9.0[^1].

Its significance outweighs its 2k stars: termenv is a foundation stone of the
Charm TUI ecosystem — charmbracelet/lipgloss and bubbletea (v1) build their
color handling on it, so a large share of modern Go TUIs transitively depend on
this library. Charm's v2 stack has been migrating the same responsibilities
into in-house packages (charmbracelet/colorprofile, charmbracelet/x/ansi),
which makes termenv's long-term role more that of a stable, standalone utility
than the ecosystem's growth center.

The defining tradeoff: termenv doesn't just parse environment variables — it
actively *queries* the terminal with OSC escape sequences (e.g. OSC 10/11 for
foreground/background color) over the TTY in raw mode. That yields answers env
vars can't give (real background color, hence `HasDarkBackground()`), but it is
also the library's main source of production surprises (see Production Notes).

## Getting Started

```bash
go get github.com/muesli/termenv
```

```go
out := termenv.NewOutput(os.Stdout)

fmt.Println("profile:", out.Profile) // Ascii | ANSI | ANSI256 | TrueColor
fmt.Println("dark background:", out.HasDarkBackground())

s := out.String("Hello World").
	Foreground(out.Color("#abcdef")). // degraded automatically per profile
	Bold()
fmt.Println(s)
```

`termenv.EnvColorProfile` additionally honors `NO_COLOR` and `CLICOLOR_FORCE`;
plain `ColorProfile` does not[^2].

## Architecture / How It Works

Detection is layered:

1. **isatty check** — if the writer isn't a TTY (pipe, CI log), the profile
   drops to Ascii and styles become no-ops.
2. **Environment heuristics** — `$TERM`, `$COLORTERM`, terminal-specific hints
   (`kitty`, Ghostty and Windows Terminal are known TrueColor; `screen`/`xterm`
   variants map to 256 colors), plus `COLORFGBG` parsing as a background
   fallback.
3. **Active OSC queries** — for background/foreground color and cursor
   position, termenv puts the controlling TTY into raw mode, emits an OSC/DSR
   query, and reads the response, giving up after a timeout and proceeding with
   defaults if the terminal never answers[^3].

Styling is a thin, chainable wrapper over ANSI SGR sequences (`Style` implements
`fmt.Stringer`), with a parallel set of `text/template` helper funcs (`Bold`,
`Color`, `Foreground`, ...). Color degradation converts RGB → nearest ANSI256 →
nearest ANSI16 via HSLuv distance, which produces perceptually better matches
than naive RGB distance[^1].

Everything hangs off an `Output` struct (introduced v0.13.0[^4]) wrapping an
`io.Writer`; package-level functions are conveniences over a default global
output. Beyond styling, `Output` exposes a wide grab-bag of terminal control:
alt-screen, scroll regions, window title, mouse tracking modes, bracketed
paste, hyperlinks, desktop notifications, and OSC 52 clipboard writes (through
aymanbagabas/go-osc52, with tmux passthrough handling)[^5]. On Windows 10+ it
enables `ENABLE_VIRTUAL_TERMINAL_PROCESSING` rather than translating sequences
to legacy console API calls.

## Production Notes

- **The OSC query is the footgun.** Background-color detection needs a
  responding terminal on a real TTY. Under `screen` and `tmux` the passthrough
  story is limited (termenv's own compatibility matrix marks color-scheme
  queries as blocked under multiplexers), inside Emacs shells it caused hangs
  until a v0.16.0 fix[^6], and on non-answering terminals you pay the fallback
  timeout. If you only need colored output, construct the output with an
  explicit profile (`termenv.WithProfile(...)`) and skip queries entirely.
- **Piped output silently loses color.** The isatty gate means CI logs and
  `| less` get Ascii. That is correct default behavior, but users will report
  it as a bug; support `CLICOLOR_FORCE` via `EnvColorProfile` to give them an
  escape hatch — and note that using bare `ColorProfile` means `NO_COLOR` is
  ignored, which some packagers treat as a compliance defect[^2].
- **Release cadence is slow and the bus factor is one.** Gaps of ~20 months
  between v0.15.2 (2023-06) and v0.16.0 (2025-02) with 48 open issues; the repo
  still receives pushes (latest 2025-11) but muesli maintains a very large
  portfolio, and Charm's own investment has shifted to its v2 internals. Treat
  termenv as stable-but-slow, not fast-moving.
- **API churn centered on output handling**: v0.13.0 introduced `Output`,
  v0.15.0 changed the default output and added `WithTTY`[^7]. New code should
  pass explicit outputs, especially in tests needing deterministic profiles.

## When to Use / When Not

**Use when:**
- You're writing a Go CLI that should print adaptive color (TrueColor where
  possible, degraded elsewhere) without hand-rolling escape codes.
- You need light/dark background detection to pick a readable palette.
- You need the odd terminal extra — OSC 52 clipboard, alt-screen, window title,
  notifications — without pulling in a full TUI framework.

**Avoid when:**
- You're building a full interactive TUI: use charmbracelet/bubbletea (which
  handles this layer for you) rather than driving mouse modes and alt-screen
  by hand through termenv.
- You just want `color.Red("error")`-style output with zero detection
  machinery: fatih/color is smaller and simpler.
- Your output primarily targets logs/CI where color is either forced or absent;
  capability probing buys nothing there.

## Alternatives

- fatih/color — simplest ANSI color printing for Go; use it when you don't need
  capability detection or degradation.
- charmbracelet/lipgloss — style-definition layer (borders, layout, alignment)
  built above this niche; use it for declarative TUI styling.
- gookit/color — 16/256/RGB color printing with tag markup; comparable scope,
  different API taste.
- charmbracelet/x — Charm's v2 internals (ansi, colorprofile packages); where
  the ecosystem's active development of this layer now happens.
- muesli/reflow — sibling library for ANSI-aware wrapping/truncation; a common
  companion, not a substitute.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2020-01-23 | Initial tagged release. |
| v0.9.0 | 2021-06-24 | HSLuv color space for profile conversions[^1]. |
| v0.11.0 | 2022-01-31 | Session commands, 5s OSC query timeout fallback, Windows VT helper[^3]. |
| v0.13.0 | 2022-09-23 | `termenv.Output` type, tmux status-report support, bracketed paste[^4]. |
| v0.15.0 | 2023-03-08 | Default output change, `WithTTY`, go-osc52 v2 for clipboard/tmux[^7]. |
| v0.16.0 | 2025-02-21 | Emacs shell hang fix, Ghostty TrueColor, uniseg-based width, z/OS build[^6]. |

## References

[^1]: termenv v0.9.0 release notes — HSLuv conversions. https://github.com/muesli/termenv/releases/tag/v0.9.0
[^2]: termenv README, `EnvColorProfile` vs `ColorProfile`. https://github.com/muesli/termenv#usage
[^3]: termenv v0.11.0 release notes — OSC timeout fallback (PR #57), Windows VT (PR #61). https://github.com/muesli/termenv/releases/tag/v0.11.0
[^4]: termenv v0.13.0 release notes — `Output` (PR #86), tmux termStatusReport (PR #85). https://github.com/muesli/termenv/releases/tag/v0.13.0
[^5]: aymanbagabas/go-osc52 — OSC 52 clipboard with tmux/screen handling. https://github.com/aymanbagabas/go-osc52
[^6]: termenv v0.16.0 release notes — Emacs hang fix (PR #152), Ghostty (PR #161). https://github.com/muesli/termenv/releases/tag/v0.16.0
[^7]: termenv v0.15.0 release notes — default output & `WithTTY` (PR #117). https://github.com/muesli/termenv/releases/tag/v0.15.0

## Tags

go, terminal, ansi, tui, cli, color, styling, escape-sequences, charm-ecosystem, library
