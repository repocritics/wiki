# wailsapp/wails

> Build desktop applications by wrapping a Go backend and a web frontend into a single native binary — using the OS's own webview instead of a bundled Chromium.

[GitHub repo](https://github.com/wailsapp/wails) ·
[Official website](https://wails.io) ·
[License: MIT](https://github.com/wailsapp/wails/blob/master/LICENSE)

## Overview

Wails is a framework for building cross-platform desktop apps where the business
logic is Go and the UI is HTML/CSS/JavaScript rendered by the operating system's
native webview[^1]. It handles project scaffolding, compilation, frontend
bundling, and Go↔JS binding, producing a single self-contained executable. The
project was created by Lea Anthony and has been in development since late 2018[^2].

Its defining decision — and its defining tradeoff — is the refusal to embed a
browser. Electron ships a full Chromium and Node runtime with every app, giving
identical rendering everywhere at the cost of ~100 MB+ binaries and high memory
use. Wails instead calls WebView2 (Chromium-Edge) on Windows, WKWebView on
macOS, and WebKitGTK on Linux[^1]. Binaries drop to a few megabytes and memory
footprint falls, but you inherit three different browser engines with three
different feature sets and rendering quirks. This is the single most important
thing to understand before choosing Wails.

The project ships two active lines: **v2** (stable, the version almost everyone
uses) and **v3** (alpha, a substantial rearchitecture toward a
multi-window/service model)[^3]. New production apps should target v2.

## Getting Started

```bash
# v2 (stable). Requires Go 1.20+, a C toolchain (CGO), and Node for the frontend.
go install github.com/wailsapp/wails/v2/cmd/wails@latest
wails doctor            # verifies platform deps (WebView2, WebKitGTK, etc.)
wails init -n myapp -t react
cd myapp
wails dev               # live-reload dev mode with native devtools
wails build             # produces a single binary in build/bin
```

```go
// app.go — a Go method exposed to JavaScript
package main

type App struct{}

func (a *App) Greet(name string) string {
    return "Hello, " + name
}
```

```js
// frontend — call the Go method; TS definitions are auto-generated
import { Greet } from "../wailsjs/go/main/App";

Greet("world").then(msg => console.log(msg)); // "Hello, world"
```

## Architecture / How It Works

A Wails app is a normal Go program that, at startup, creates a native window
containing a webview and serves the compiled frontend assets to it (embedded via
Go's `embed` at build time). Communication is bidirectional:

- **Bound methods** — Go structs registered in `wails.Run(...)` are reflected
  into JavaScript as async functions. Wails generates matching TypeScript
  definitions from the Go signatures, so struct fields and method params are
  typed on the JS side[^1].
- **Events** — a unified pub/sub bus lets Go emit named events to JS and vice
  versa, used for pushing updates the frontend didn't request.
- **Runtime** — a JS/Go API for native concerns: window controls, dialogs,
  menus, clipboard, dark/light mode, and (on supported platforms) translucency
  and frosted-window effects.

Because the UI is a native webview, **the rendering engine is not part of your
app**. Windows uses the evergreen WebView2 runtime; macOS uses the system
WKWebView tied to the OS version; Linux uses whatever WebKitGTK the distro
provides. There is no bundled engine to pin, which is why the same frontend can
render differently across platforms.

Building requires **CGO**, because Wails binds to native windowing and webview
APIs (WebView2 COM interfaces, Cocoa/WKWebView, GTK). This has a large downstream
consequence: cross-compilation is difficult. You generally build on each target
OS (or in a CI matrix / containers), not with `GOOS=windows go build` from a Mac.

v3 (alpha) reworks this model around an application/services abstraction with
first-class multi-window support and a redesigned runtime, but the API is not
stable and it is not recommended for shipping software yet[^3].

## Production Notes

- **Cross-platform rendering is not free.** WebView2, WKWebView, and WebKitGTK
  diverge on CSS support, font rendering, and JS API availability. WKWebView in
  particular lags Chromium on some newer web features, and Safari-family quirks
  apply. Test on every target; do not assume "works in Chrome" means "works in
  the app."
- **Linux is the hardest target.** WebKitGTK versions vary widely across distros
  (webkit2gtk-4.0 vs 4.1, differing patch levels), producing rendering and
  hardware-acceleration differences. Some environments require disabling GPU
  acceleration (`WEBKIT_DISABLE_COMPOSITING_MODE`/DMABUF workarounds) to avoid a
  blank or garbled window. Packaging must declare the right WebKitGTK dependency.
- **Windows needs the WebView2 runtime.** Modern Windows 10/11 ship it, but you
  must choose an install strategy (evergreen bootstrapper vs embedded fixed
  version) for machines that lack it.
- **CGO cross-compilation.** Plan a per-OS build pipeline. Expect friction using
  standard Go cross-compile flows; native toolchains for each target are the
  reliable path.
- **Binary size and memory** are dramatically lower than Electron, but you are
  trading that for the maintenance cost of three engines. If you have never
  shipped for all three desktop OSes, budget real QA time.
- **v2 → v3 is not a drop-in upgrade.** v3 is a rearchitecture; treat a move to
  it as a migration, and only after it reaches stable.

## When to Use / When Not

**Use when:**
- Your backend logic is (or wants to be) Go and you want a web UI without running
  an HTTP server and a browser.
- Small binary size and low memory matter more than pixel-identical rendering.
- You want native menus, dialogs, and window chrome with a familiar web frontend
  (React/Vue/Svelte/Angular/vanilla all work).

**Avoid when:**
- You need guaranteed identical rendering across platforms — Electron's bundled
  Chromium is the safer choice.
- You must cross-compile from a single machine with minimal build infrastructure.
- You want mobile targets today, or a non-Go backend — look at Tauri.
- Your Linux support matrix is broad and you can't absorb WebKitGTK variance.

## Alternatives

- tauri-apps/tauri — same system-webview philosophy with a Rust core; larger
  ecosystem, plugin system, and mobile targets. Use instead when you prefer Rust
  or need iOS/Android.
- electron/electron — bundles Chromium + Node. Use instead when consistent
  rendering across platforms outweighs binary size and memory.
- webview/webview — a tiny C library with Go/other bindings and no framework. Use
  instead when you want just a webview window, not scaffolding and tooling.
- fyne-io/fyne — pure-Go, canvas-rendered native widget toolkit (no web tech at
  all). Use instead when you want an all-Go UI and no webview engines.
- neutralinojs/neutralinojs — lightweight, language-agnostic system-webview
  framework. Use instead when you don't want a Go backend.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo start | 2018-12 | Initial commit; "Webview on Rails" concept[^2]. |
| v1 | 2019 | First stable line; single-engine-per-OS webview approach[^2]. |
| v2 (beta) | 2021 | Ground-up rewrite: new runtime, binding model, TS gen[^1]. |
| v2.0 | 2022-09 | v2 stable release; the current default line[^1]. |
| v3 (alpha) | 2023– | Rearchitecture: app/services model, multi-window[^3]. |

## References

[^1]: Wails v2 documentation and introduction. https://wails.io/docs/introduction
[^2]: Wails README — project origin ("Webview on Rails") and author. https://github.com/wailsapp/wails#faq
[^3]: Wails v3 alpha documentation. https://v3.wails.io/

## Tags

go, golang, desktop-app, webview, cross-platform, electron-alternative, gui, native-webview, single-binary, frontend
