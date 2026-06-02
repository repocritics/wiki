# tauri-apps/tauri

Tauri — a Rust-based framework for building tiny, secure, cross-platform desktop apps with a web frontend. Electron alternative shipping ~10MB binaries vs. Electron's ~150MB.

## What it is

A Rust framework that pairs system-WebView-based rendering (no bundled Chromium) with a Rust backend. The frontend can be any web framework (React, Vue, Svelte, Angular, Solid, Astro); the Rust core handles native API calls + IPC. Apps ship as small native binaries because they reuse the OS WebView (WebView2 on Windows, WebKit on macOS, WebKitGTK on Linux). Tauri 2.0 added mobile support (iOS, Android).

## Key features

- ~10MB app binaries (vs. ~150MB for Electron).
- Rust backend + any web frontend (via web view).
- Cross-platform desktop (Win, macOS, Linux) + mobile (iOS, Android in Tauri 2).
- Permission system for fine-grained API access.
- Auto-updater, bundler, signing all in tauri CLI.
- MIT / Apache-2.0 dual-licensed.

## Tech stack

- Rust on the backend.
- Web frontend in any framework.
- System WebView for rendering.

## When to reach for it

- You want a small desktop app binary.
- You're a web team with Rust comfort, wanting native desktop without Electron's bloat.
- You're shipping to mobile + desktop from one Rust core.

## When *not* to reach for it

- You want full Chromium consistency — Electron renders the same Chromium everywhere; Tauri renders per-OS WebView.
- You don't want Rust in your stack.

## Maturity signal

Actively maintained under Tauri Foundation. Tauri 2.0 launch added mobile support, expanding scope significantly.

## Alternatives

- Electron — for Chromium-everywhere consistency.
- Wails — Go-flavored equivalent.
- Neutralinojs — JS-flavored lightweight.

## Tags

rust, desktop, cross-platform, framework, electron-alternative, mit-license, apache-license, mobile
