# DioxusLabs/dioxus

> A Rust framework for building web, desktop, and mobile apps from a single codebase, using a React-like component model and a renderer-agnostic virtual DOM.

[GitHub repo](https://github.com/DioxusLabs/dioxus) ·
[Official website](https://dioxuslabs.com) ·
[License: MIT OR Apache-2.0](https://github.com/DioxusLabs/dioxus/blob/main/LICENSE-APACHE)

## Overview

Dioxus is a Rust UI framework whose surface is deliberately React-shaped: components are functions returning an `Element`, markup is written in an `rsx!` macro, and a virtual DOM diffs renders into mutations. What differs is the substrate — everything is Rust, compiled ahead of time, and the same component tree can be driven onto several renderers (WebAssembly/DOM, native webview, server-side HTML, LiveView, or an experimental GPU renderer) without rewriting the UI code[^1].

State management moved from a hooks-only model to a **signals** model in the 0.5 release[^2], borrowing fine-grained reactivity ideas from SolidJS and Svelte while keeping React's component ergonomics. `use_signal` returns a `Copy` handle that can be moved into closures freely — a design choice that sidesteps much of Rust's borrow-checker friction in event handlers, at the cost of some runtime bookkeeping. The result reads more like Leptos than like Yew.

The defining tension is maturity versus ambition. Dioxus targets an unusually wide platform matrix (web, desktop, iOS, Android, server, plus experimental native rendering) with a single API, and ships its own `dx` CLI, bundler, and subsecond Rust hot-patching. All of this is pre-1.0: the framework is still making breaking changes every minor release, and several headline features (the WGPU/Blitz native renderer, hot-patching) are explicitly experimental. It is funded and staffed by a small full-time core team backed by FutureWei, Satellite.im, and the GitHub Accelerator program[^3].

## Getting Started

```bash
cargo install dioxus-cli --locked      # provides the `dx` command
# or: curl -fsSL https://dioxuslabs.com/install.sh | bash
dx new my-app && cd my-app
dx serve                               # dev server + hot reload
```

```rust
use dioxus::prelude::*;

fn main() {
    dioxus::launch(app);
}

fn app() -> Element {
    let mut count = use_signal(|| 0);

    rsx! {
        h1 { "High-Five counter: {count}" }
        button { onclick: move |_| count += 1, "Up high!" }
        button { onclick: move |_| count -= 1, "Down low!" }
    }
}
```

The same `app` compiles to web (`dx serve --platform web`), desktop, or mobile (`dx serve --platform android`) by changing a target rather than the source.

## Architecture / How It Works

The core (`dioxus-core`) is a platform-independent virtual DOM. `rsx!` expands at compile time into templates: static structure is hoisted into a template that the renderer instantiates once, and only the dynamic nodes are diffed on re-render. The diff produces a stream of `Mutation`s (create/remove/set-attribute/etc.) that a renderer applies to its own backing store. This indirection is what lets one component tree drive many targets:

- **Web** — `dioxus-web` applies mutations directly to the browser DOM via `web-sys`, compiled to WebAssembly. A "hello world" is roughly 50 kb, comparable to React[^1].
- **Desktop / Mobile** — `dioxus-desktop` renders into a system webview (wry/tao under the hood, the same stack Tauri uses). The UI is HTML/CSS in a webview; Rust runs natively and calls into the page. This is *not* native-widget rendering.
- **SSR / Fullstack** — `dioxus-ssr` renders to an HTML string; `dioxus-fullstack` adds hydration, suspense, and **server functions** (annotated Rust functions callable from the client as RPC), integrated with the axum web framework[^4].
- **LiveView** — renders on the server and streams DOM patches over a WebSocket, Phoenix-LiveView style.
- **Native (experimental)** — **Blitz** renders HTML/CSS without a webview, using Servo's `stylo` CSS engine and a WGPU/Vello paint layer. This is the path toward true native rendering and is not production-ready.

Reactivity is signal-based: reads inside a component subscribe that component (or an `Effect`/`Memo`) to the signal, and writes mark subscribers dirty for the next render. Because signal handles are `Copy`, they cross closure boundaries without clone/move gymnastics — the ergonomic payoff over Yew's `Rc`-heavy patterns.

The `dx` CLI is a first-party part of the story, not an afterthought: it owns the dev server, asset pipeline (`.avif` generation, WASM compression, minification), bundling to `.app`/`.ipa`/`.apk`, and the hot-patching engine.

## Production Notes

**Pre-1.0 churn is the dominant operator cost.** Minor releases carry breaking changes and non-trivial migrations. The 0.4→0.5 jump replaced the state-management model (hooks → signals), and 0.5→0.6→0.7 each moved APIs and CLI behavior. Pin exact versions and budget for migration work on every upgrade; treat blog release notes as required reading, not optional.

**Compile times and WASM size.** This is Rust plus a proc-macro-heavy UI layer, so cold builds are slow and incremental UI iteration relies on hot-reload rather than full recompiles. `dx serve` hot-reloads `rsx!` markup and assets in milliseconds; `dx serve --hotpatch` extends this to Rust code changes but is experimental. For web, ship builds go through `wasm-opt` and compression — measure the delivered `.wasm`, not the debug artifact.

**Desktop is a webview, with the tradeoffs that implies.** Memory footprint and rendering fidelity track the host's webview (WebView2 on Windows, WebKit on macOS/Linux), and Linux WebKitGTK is the usual source of platform-specific rendering and packaging bugs. If you need webview-free native rendering today, Dioxus is not there yet — Blitz is experimental.

**Ecosystem depth.** The component and crate ecosystem (dioxus-community org, `dioxus-std`/SDK, first-party primitives modeled on shadcn/ui and Radix) is real but far smaller than React's. Expect to write integrations that would be off-the-shelf in JavaScript, and to interact with native platform APIs (JNI, Objective-C, web-sys) directly for anything the framework doesn't wrap.

**Fullstack coupling to axum.** Server functions and the fullstack renderer assume axum; using another server framework means dropping to the lower-level pieces. Hydration mismatches between server and client render are the usual class of subtle bug, as in any SSR framework.

## When to Use / When Not

**Use when:**
- You want one Rust codebase spanning web, desktop, and mobile with a React-like API.
- Your team is already invested in Rust and wants type-safe fullstack with server functions.
- You want the CLI, bundler, and hot-reload provided rather than assembled.
- Webview-based desktop/mobile output (Tauri-class) is acceptable.

**Avoid when:**
- You need long-term API stability now — pre-1.0 breaking changes will cost you on every upgrade.
- You need webview-free native rendering in production (Blitz is experimental).
- The team lacks Rust experience and needs prototype velocity — JS/TS frameworks win.
- You need a large mature component/library ecosystem out of the box.

## Alternatives

- leptos-rs/leptos — Rust fullstack with fine-grained signals and server functions; no virtual DOM, web-first. Use instead when you want maximal web performance and are less focused on native desktop/mobile.
- yewstack/yew — older Rust/WASM framework, web-only, `Rc`-based state. Use instead when you want a longer-established web-only option.
- tauri-apps/tauri — Rust backend + web-frontend (any JS framework) desktop/mobile shell. Use instead when your UI is already JS/TS and you only need a native wrapper.
- emilk/egui — immediate-mode Rust GUI rendered on GPU, no HTML/CSS. Use instead for tools/games UIs where native rendering matters more than web reach.
- slint-ui/slint — declarative native UI with its own markup language and toolchain. Use instead when you want native-widget rendering and a designer-oriented DSL.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2022 | First public release; virtual DOM + `rsx!`, web and desktop. |
| 0.3 | 2023 | Broader platform work, CLI improvements. |
| 0.4 | 2023 | Mobile support and hot-reloading advances. |
| 0.5 | 2024 | Signals-based state management replaces hooks-only model[^2]. |
| 0.6 | 2024 | CLI overhaul, docs and fullstack improvements. |
| 0.7 | 2025–2026 | Subsecond hot-patching, first-party components, Blitz native renderer, axum-integrated fullstack[^1]. |

## References

[^1]: Dioxus website and repository README — features, platforms, and 0.7 tour. https://dioxuslabs.com/learn/0.7/
[^2]: Dioxus blog, "Dioxus 0.5" — signals-based reactivity. https://dioxuslabs.com/blog/release-050
[^3]: Dioxus README, "Full-time core team" — FutureWei, Satellite.im, GitHub Accelerator backing. https://github.com/DioxusLabs/dioxus
[^4]: axum web framework (tokio-rs), the fullstack integration target. https://github.com/tokio-rs/axum

## Tags

rust, wasm, cross-platform, ui-framework, virtual-dom, signals, desktop, mobile, ssr, fullstack, webview, reactive
