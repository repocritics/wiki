# electron/electron

The framework that brought Chromium + Node.js together into a single runtime — powers cross-platform desktop apps (VS Code, Slack, Discord, Figma desktop, 1Password, and many others) from a JavaScript/HTML/CSS codebase.

## What it is

A runtime that ships Chromium (for rendering) and Node.js (for system access) as a single embeddable engine. Apps write their UI as web pages and their privileged logic as Node.js modules, with a main/renderer process split for security. The trade-off — large binaries and per-app Chromium copies — is the recurring complaint; the gain — one codebase across Windows, macOS, and Linux with full DOM/CSS/JS — is why the framework remains dominant for "desktop app I want shipped this year".

## Key features

- Chromium + Node.js bundled into one runtime, exposed via main + renderer processes.
- Built-in IPC primitives for renderer ↔ main communication.
- Auto-updater, crash reporter, native menus, system tray, notifications all in standard library.
- Native module compilation toolchain for project-specific C++ integrations.
- electron-builder / electron-forge are the canonical app-shipping toolchains.
- MIT-licensed.

## Tech stack

- C++ primary (Chromium glue + Node integration layer).
- JavaScript / TypeScript at the app surface — that's the developer-facing API.
- V8 for JavaScript execution; Node.js native modules.
- Released on a Chromium-tracking cadence — each Electron major version pins to a specific Chromium and Node version pair.

## When to reach for it

- You need a cross-platform desktop app shipped on a JS/web team's existing skills.
- You want full web-platform features (CSS animations, modern JS, all of npm) on the desktop.
- Your app's bottleneck is feature velocity, not memory footprint.

## When *not* to reach for it

- Memory footprint matters — Electron apps idle at 100MB+ and scale up from there.
- You need true native UI (NSWindow / Win32 / GTK widgets) — Electron renders DOM, not native widgets.
- You want a smaller binary — Tauri (with system WebView + Rust backend) ships ~10MB vs. Electron's ~150MB+.

## Maturity signal

121k stars, 17k forks, MIT, last push the day this page was generated. 13-year-old project under the OpenJS Foundation. Open-issues count of 863 is moderate for a project of this size; the team triages aggressively. Release cadence is locked to Chromium's, so security patches land predictably.

## Alternatives

- `tauri-apps/tauri` — use when you want smaller binaries and a Rust backend instead of Node.
- Flutter Desktop — use when you want pixel-identical UI across mobile + desktop in Dart.
- Native frameworks (SwiftUI on macOS, WinUI on Windows) — use when full native fidelity matters per-platform.
- Web app + PWA — use when you can ship in a browser instead of a desktop binary.

## Notes

The "Electron is bloated" critique is real (each app ships its own Chromium) and partially addressed by the OS-bundled WebView2 / WebKit alternatives that Tauri exploits. License (MIT) and OpenJS Foundation governance make Electron the safest choice for vendor-stable desktop development; the cost is the binary size and per-app memory hit.

## Tags

electron, javascript, typescript, c-plus-plus, chromium, nodejs, desktop, cross-platform, framework, app-framework
