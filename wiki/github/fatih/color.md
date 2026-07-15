# fatih/color

> ANSI color for Go terminal output, with a legacy-Windows translation layer bundled in.

[GitHub repo](https://github.com/fatih/color) ·
[pkg.go.dev](https://pkg.go.dev/github.com/fatih/color) ·
[License: MIT](https://github.com/fatih/color/blob/main/LICENSE.md)

## Overview

`color` wraps ANSI escape codes[^1] behind a small Go API so that CLI output can
carry foreground/background colors and attributes (bold, underline) without the
caller hand-writing escape sequences. It has existed since 2014[^2] and is one of
the most depended-upon leaf packages in the Go CLI ecosystem — pulled in
transitively by a large share of Go command-line tools. It is mature rather than
active: the last pushes are recent (2026-07), but the cadence is slow and the API
surface has been stable for years. ~8k stars and 640 forks reflect ubiquity, not
churn; 25 open issues is low for its reach.

The package's defining choice is convenience over purity. It ships package-level
globals (`color.NoColor`, `color.Output`, `color.Error`) and helper functions
(`color.Red(...)`, `color.Set(...)`) so the simplest use is one line. That same
global state is the source of most of its sharp edges: enable/disable is a process
-wide mutable flag, and the "am I writing to a TTY?" decision is made once, based
on stdout, at package init. For a short-lived CLI this is exactly right. For a
long-running server, a library, or concurrent writers, the globals leak across
boundaries you did not intend.

The second defining choice is the bundled Windows story. On legacy Windows
consoles that do not interpret ANSI, `color` routes through `mattn/go-colorable`
to translate escape sequences into Win32 console API calls, and uses
`mattn/go-isatty` for TTY detection[^3][^4]. That is why it "supports Windows too"
where a raw `fmt.Println("\x1b[31m...")` would print garbage on old shells.

## Getting Started

```
go get github.com/fatih/color
```

```go
package main

import "github.com/fatih/color"

func main() {
	// One-shot helpers (newline appended automatically)
	color.Cyan("Prints %s in cyan.", "text")

	// Reusable color object with combined attributes
	warn := color.New(color.FgYellow, color.Bold)
	warn.Println("bold yellow warning")

	// Insert colored fragments into an otherwise plain string
	red := color.New(color.FgRed).SprintFunc()
	println("status: " + red("FAILED"))

	// 24-bit color (only if the terminal actually supports truecolor)
	color.RGB(255, 128, 0).Println("orange")
}
```

## Architecture / How It Works

The core type is `Color`, a list of `Attribute` values (`FgRed`, `Bold`,
`BgWhite`, …) where each `Attribute` is an `int` mapping to an SGR parameter.
`Color.wrap(s)` prepends the assembled `\x1b[...m` sequence and appends the reset
`\x1b[0m`. Everything else — `Print`, `Printf`, `Sprint`, `Fprint`, and the
`*Func` factories (`PrintfFunc`, `SprintFunc`, …) — is sugar over that wrap.

Output goes through two package-level `io.Writer`s: `color.Output` (stdout,
colorable-wrapped) and `color.Error` (stderr). The helper functions and `Set`/
`Unset` write to these globals; that is the mechanism that makes Windows
translation transparent, and also why writing colored text via bare `fmt.Println`
instead of `color.Output` will not translate on legacy Windows.

Color enablement is governed by `color.NoColor`, a package `bool` computed at init
from three inputs: whether stdout is a TTY (`go-isatty`), and the `NO_COLOR`
environment variable[^5]. If `NO_COLOR` is set to any non-empty value, output is
plain. A single `Color` object can also override locally via `DisableColor()` /
`EnableColor()`, which pins that object's behavior independent of the global.

`Set`/`Unset` implement stateful coloring for code you do not control: `Set`
writes the escape prefix to `color.Output` with no reset, so all following writes
inherit the color until `Unset` (typically `defer`red) emits the reset. This is a
side-channel on shared global output, not a scoped context.

## Production Notes

**Global mutable state is not concurrency-safe to toggle.** `color.NoColor`,
`color.Output`, and the `Set`/`Unset` pair are process globals. Flipping
`NoColor` or interleaving `Set`/`Unset` from multiple goroutines is a data race
and produces interleaved escape sequences. Building distinct `*Color` objects and
using `SprintFunc`/`Fprint` to a specific writer is the concurrency-safe pattern.

**TTY detection is decided once, at init, from stdout.** If your program
reassigns `os.Stdout`, writes to a pipe you opened after startup, or is a library
whose caller redirects output, `NoColor` may not reflect the real destination.
You then set `color.NoColor` manually — which is itself a global write.

**No terminal-capability downsampling.** `color.RGB(...)` emits 24-bit truecolor
escapes unconditionally; the package does not detect whether the terminal supports
truecolor and does not fall back to 256- or 16-color approximations. On terminals
without truecolor you get wrong colors or literal artifacts. If you need capability
detection and graceful downgrade, `muesli/termenv` is the standard answer.

**`NO_COLOR` yes, `FORCE_COLOR` semantics are manual.** The package honors the
`NO_COLOR` convention but does not implement a `FORCE_COLOR` counterpart; forcing
color (e.g. for CI logs that do accept ANSI) means explicitly setting
`color.NoColor = false`, as the README's GitHub Actions note describes.

**`Set` without `Unset` bleeds.** Because `Set` emits no reset, a forgotten or
skipped `Unset` (early return, panic without `defer`) colors the rest of the
stream. Prefer scoped `*Color` objects unless you specifically need the
plug-into-existing-`fmt`-code behavior.

**Windows translation adds a dependency edge.** `go-colorable` and `go-isatty`
come along transitively. On Windows 10+ with virtual-terminal processing enabled,
native ANSI works and the translation is a no-op; the layer matters mainly for
older consoles.

## When to Use / When Not

**Use when:**
- You are writing a Go CLI and want colored output in one or two lines.
- You need legacy-Windows console support without wiring it yourself.
- You want `NO_COLOR` / non-TTY auto-disable handled for free.
- Basic 16/256-color output (and optional truecolor for known terminals) is enough.

**Avoid when:**
- You need terminal capability detection and automatic truecolor→256→16 downsampling — use termenv.
- You are building styled layouts (boxes, tables, adaptive light/dark) — use lipgloss.
- You color from many goroutines and want to avoid global state — construct `*Color` objects and write to explicit writers, or pick a global-free library.
- You want a zero-dependency package — this pulls in the mattn Windows helpers.

## Alternatives

- muesli/termenv — color profile detection and truecolor downsampling; the base most Charm tooling builds on. Use when the terminal's real capability matters.
- charmbracelet/lipgloss — declarative styling and layout (borders, alignment, adaptive colors). Use when you are styling UIs, not just coloring words.
- gookit/color — comparable ergonomics with richer 256/truecolor and tag-based styling. Use when you want more built-in features in one package.
- logrusorgru/aurora — chainable, expression-style coloring. Use when you prefer `au.Bold(au.Red("x"))` composition.
- mgutz/ansi — minimal helper that just returns ANSI-wrapped strings. Use when you want the codes and nothing else.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-02-17 | First commit; ANSI helpers + Windows support via go-colorable[^2]. |
| — | ~2019 | `NO_COLOR` environment-variable support added[^5]. |
| v1.16.0 | ~2023-12 | 24-bit truecolor API (`RGB`, `BgRGB`, `AddRGB`, `AddBgRGB`). |

## References

[^1]: ANSI escape code (Colors). https://en.wikipedia.org/wiki/ANSI_escape_code#Colors
[^2]: fatih/color repository (created 2014-02-17). https://github.com/fatih/color
[^3]: mattn/go-colorable — ANSI-to-Win32 console translation. https://github.com/mattn/go-colorable
[^4]: mattn/go-isatty — TTY detection. https://github.com/mattn/go-isatty
[^5]: NO_COLOR informal standard. https://no-color.org

## Tags

go, golang, cli, terminal, ansi, color, tty, windows, console, output-formatting
