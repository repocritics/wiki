# fyne-io/fyne

> A cross-platform GUI toolkit in Go that renders its own widgets with OpenGL, trading native look for a single codebase across desktop and mobile.

[GitHub repo](https://github.com/fyne-io/fyne) ·
[Official website](https://fyne.io/) ·
[License: BSD-3-Clause](https://github.com/fyne-io/fyne/blob/develop/LICENSE)

## Overview

Fyne is a GUI toolkit written in Go, first released as v1.0 in March 2019[^1] and led
primarily by Andrew Williams. It targets one specific promise: write the UI once in Go
and ship it to Linux, macOS, Windows, BSD, Android, and iOS from the same source. Its
visual language is inspired by Material Design, and it ships a themeable widget set,
canvas primitives, layout containers, and data binding. As of 2026 it is the
most-starred Go-native GUI project (~28.5k stars), actively developed on the `develop`
branch with a monthly-to-quarterly release cadence — v2.8.0 landed in July 2026[^2].

The defining architectural choice — and the source of every tradeoff below — is that
Fyne does not use the host operating system's native widgets. It draws every button,
label, and list itself onto an OpenGL surface. This buys pixel-identical rendering
across platforms and full control over theming, at the cost of not looking or behaving
like a native app, and a limited accessibility story (there is no OS accessibility tree
to hook into because the OS sees only a GL canvas).

The second defining tension is CGO. Despite "written in Go" branding, Fyne depends on C
bindings for OpenGL and GLFW on desktop, so a C compiler is mandatory and pure-Go
cross-compilation (`GOOS=windows go build` from Linux) does not work out of the box.

## Getting Started

Requires Go 1.17+, a C compiler, and your platform's development/graphics headers.

```bash
go get fyne.io/fyne/v2@latest
go mod tidy
```

```go
package main

import (
	"fyne.io/fyne/v2/app"
	"fyne.io/fyne/v2/container"
	"fyne.io/fyne/v2/widget"
)

func main() {
	a := app.New()
	w := a.NewWindow("Hello")

	hello := widget.NewLabel("Hello Fyne!")
	w.SetContent(container.NewVBox(
		hello,
		widget.NewButton("Hi!", func() {
			hello.SetText("Welcome :)")
		}),
	))

	w.ShowAndRun() // blocks: runs the render/event loop on the main goroutine
}
```

Run with `go run main.go`. For distributable bundles (icons, metadata, mobile
packaging) install the CLI: `go install fyne.io/tools/cmd/fyne@latest`, then
`fyne package -os android -appID my.domain.app`.

## Architecture / How It Works

Everything visible is a `fyne.CanvasObject`. Primitives (`canvas.Rectangle`, `Text`,
`Image`, `Line`, `Circle`) are the leaves; widgets are composites that expose a
`WidgetRenderer` describing which primitives to draw and how to lay them out. Layout is
handled by container layouts (`VBox`, `HBox`, `Grid`, `Border`, `Max`) that implement a
`Layout` interface, not by CSS or constraint solving.

Rendering goes through a **driver** abstraction. `driver/glfw` backs desktop (GLFW for
windowing, OpenGL for painting); `driver/mobile` backs Android and iOS via
`golang.org/x/mobile` with OpenGL ES; and a `test` driver plus a software renderer let
you run the full widget tree headless in unit tests. This last piece is genuinely
useful — you can assert on rendered output without a display server.

Theming is centralized: a `fyne.Theme` supplies colors, fonts, sizes, and icons, and
every widget reads from it, so a single theme swap restyles the whole app and light/dark
switching is automatic. Data binding (`data/binding`, added in v2.0) provides observable
values that widgets subscribe to, reducing manual `SetText`/`Refresh` wiring.

The toolkit is intentionally opinionated and self-contained: layouts, widgets, storage
abstraction (`fyne.io/fyne/v2/storage`), preferences, notifications, and system tray are
all in-tree rather than assembled from third-party libraries. The cost of that
cohesion is that extending beyond the built-in widget set means implementing
`WidgetRenderer` by hand, and there is no escape hatch to drop in a native control.

## Production Notes

- **CGO makes cross-compilation the main operational pain.** Because desktop builds link
  OpenGL/GLFW C code, you cannot simply `GOOS`/`GOARCH` your way to other platforms. The
  community answer is `fyne-cross`[^3], a Docker-based tool that runs the target
  toolchains in containers. Budget CI time for it; native-per-platform runners are often
  simpler than one cross-building host.
- **First Windows build is slow.** Compiling the GLFW/OpenGL C dependencies can take
  several minutes on the first build (the README notes up to ~10 minutes on some
  hardware); subsequent incremental builds are fast.
- **CJK and emoji fonts are a recurring footgun.** The bundled default font does not
  cover Chinese/Japanese/Korean or emoji, so non-Latin text renders as blank/tofu until
  you supply a font via a custom theme or the `FYNE_FONT` environment variable. This is
  one of the most-filed classes of issue.
- **Not native = accessibility and platform-integration gaps.** Screen readers, native
  right-click menus, and OS-level text services see a GL surface, not widgets. If
  accessibility compliance is a hard requirement, Fyne is a poor fit.
- **Binary size and GPU dependency.** Apps embed GL bindings and ship as multi-megabyte
  binaries, and they require a working OpenGL (or GL ES) driver at runtime — headless
  servers and some minimal/virtualized environments need a software GL like Mesa/llvmpipe.
- **Threading.** Canvas/widget mutations are safe to call from goroutines (Fyne marshals
  them to the render thread), but long work on the main goroutine will stall the UI —
  keep heavy work off the callback path and let bindings push results back.
- **Upgrade note.** v1 → v2 (Jan 2021) changed the import path to `fyne.io/fyne/v2` and
  reworked APIs; there is no in-place upgrade from v1. Within v2 the API has stayed
  compatible across the 2.x line.

## When to Use / When Not

**Use when:**
- You want one Go codebase across desktop and mobile and control your own visual style.
- Your team is Go-first and wants to avoid a JS/web frontend or a C++ toolchain.
- Consistent cross-platform appearance matters more than matching each OS's native look.
- You value an all-in-one, batteries-included toolkit over assembling libraries.

**Avoid when:**
- Native look-and-feel or OS accessibility compliance is a requirement.
- You need pure-Go, CGO-free builds or trivial cross-compilation.
- You need a large/complex widget set (advanced data grids, docking, rich text editors)
  beyond what the built-in widgets provide.
- You're building a web app — a browser or webview-based stack fits better.

## Alternatives

- wailsapp/wails — use when your team prefers building the frontend in HTML/CSS/JS with a Go backend instead of Go-native widgets.
- gioui/gio — use when you want an immediate-mode design and more direct GPU control, and want to minimize the retained widget tree.
- andlabs/ui — use when native OS widgets and accessibility matter more than styling control (note: limited maintenance).
- therecipe/qt — use when you need Qt's mature, native-looking widget set and accept a heavy C++/Qt toolchain.
- webview/webview — use when you just need to render an existing web app inside a lightweight native window.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019-03-19 | First stable release; OpenGL canvas renderer[^1]. |
| 1.1.0 | 2019-07-01 | Widget and layout expansion. |
| 1.3.0 | 2020-06-05 | Mobile support maturing, new widgets. |
| 1.4.0 | 2020-11-04 | Collection widgets (List, Table, Tree). |
| 2.0.0 | 2021-01-25 | Import path `fyne.io/fyne/v2`; data binding, animations, storage rework[^4]. |
| 2.3.0 | 2022-12-24 | Theme and metadata improvements. |
| 2.4.0 | 2023-09-01 | Rich text, more widgets, layout additions. |
| 2.5.0 | 2024-07-14 | Rendering and mobile refinements. |
| 2.6.0 | 2025-04-10 | Continued widget/theme work. |
| 2.8.0 | 2026-07-11 | Latest release on the 2.x line[^2]. |

## References

[^1]: Fyne v1.0.0 release, 2019-03-19. https://github.com/fyne-io/fyne/releases/tag/v1.0.0
[^2]: Fyne v2.8.0 release, 2026-07-11. https://github.com/fyne-io/fyne/releases/tag/v2.8.0
[^3]: fyne-cross — Docker-based cross-compilation tool. https://github.com/fyne-io/fyne-cross
[^4]: Fyne v2.0.0 release, 2021-01-25. https://github.com/fyne-io/fyne/releases/tag/v2.0.0

## Tags

go, golang, gui, gui-toolkit, cross-platform, desktop, mobile, opengl, material-design, cgo, ui-framework
