# yewstack/yew

> A Rust framework for building client-side web apps that compile to WebAssembly, with a JSX-like macro and a component model.

[GitHub repo](https://github.com/yewstack/yew) ·
[Official website](https://yew.rs) ·
[License: Apache-2.0](https://github.com/yewstack/yew/blob/master/LICENSE-APACHE)

## Overview

Yew is a front-end web framework written in Rust that runs in the browser via WebAssembly[^1]. You write components in Rust, describe their markup with an `html!` macro that looks like JSX, and the whole app is compiled to a `.wasm` bundle plus a small JavaScript loader. It targets the same problem space as React — single-page apps with a component tree and a virtual DOM — but keeps the entire application logic in Rust, calling into the DOM through `wasm-bindgen` and `web-sys`.

Yew is one of the oldest and most-starred entries in the Rust/Wasm front-end space (first releases date to 2017–2018, originally authored by Denis Kolodin). Its early design borrowed from Elm: a component owned state, received `Message`s, and returned a new view from an `update`/`view` pair. Since version 0.19 (2021) the framework pivoted toward a React-style model with **function components and hooks** (`use_state`, `use_effect`, `use_context`, `use_reducer`), and that is now the recommended way to write Yew code[^2]. The older struct-component API still exists and is still used for stateful, lifecycle-heavy components.

The defining tension in Yew is that it is a **virtual-DOM framework in an ecosystem that has since moved toward fine-grained reactivity**. Newer Rust/Wasm frameworks (Leptos, Sycamore, Dioxus) largely abandon the diff-the-whole-tree model in favor of signals that update only the DOM nodes that changed. Yew's diffing approach is familiar and battle-tested, but it is not the performance leader in its own language's ecosystem, and it has never reached a 1.0 release — the API is still on 0.x and has broken across minor versions.

## Getting Started

Yew apps are typically built with **Trunk**, a Wasm web bundler[^3]:

```bash
cargo install trunk
rustup target add wasm32-unknown-unknown
cargo new my-app && cd my-app
cargo add yew --features csr
```

A minimal function component with state:

```rust
use yew::prelude::*;

#[function_component(App)]
fn app() -> Html {
    let count = use_state(|| 0);
    let onclick = {
        let count = count.clone();
        move |_| count.set(*count + 1)
    };

    html! {
        <div>
            <p>{ format!("Count: {}", *count) }</p>
            <button {onclick}>{ "Increment" }</button>
        </div>
    }
}

fn main() {
    yew::Renderer::<App>::new().render();
}
```

With an `index.html` next to `Cargo.toml`, `trunk serve` compiles to Wasm and serves with live reload; `trunk build --release` produces the deployable bundle.

## Architecture / How It Works

**Compilation target.** Yew compiles to `wasm32-unknown-unknown`. There is no Node, no JS runtime of your own code — the browser loads a `.wasm` module and a `wasm-bindgen`-generated JS shim that bridges Rust and Web APIs. All DOM access ultimately goes through `web-sys` (raw Web IDL bindings) and `gloo` (higher-level utilities).

**The `html!` macro.** `html!` is a procedural macro that parses JSX-like syntax at compile time into a tree of `VNode` values (`VTag`, `VText`, `VComp`, `VList`, `VPortal`). Because it is a macro, malformed markup and many type errors are caught at compile time rather than at runtime — but macro errors can be cryptic, and IDE support inside `html!` is weaker than for ordinary Rust.

**Rendering.** On each render Yew builds a new virtual DOM subtree and diffs it against the previous one, applying the minimal set of real DOM mutations. This is the classic React-style reconciliation. It means a component re-render walks its whole subtree to compute a diff, in contrast to signal-based frameworks that surgically update individual nodes.

**Component models.** Two coexist:
- **Function components** (`#[function_component]`) with hooks — the modern default. State lives in hooks; re-render is triggered when a `use_state`/`use_reducer` handle is updated.
- **Struct components** implementing the `Component` trait with `create`/`update`/`view`/`changed`/`rendered` lifecycle methods and an explicit `Message` enum. More verbose, but gives precise control and is still preferred for complex stateful widgets.

**Agents / workers.** Yew's `yew-agent` crate runs Rust code in Web Workers for off-main-thread computation, reflecting the framework's long-standing "multi-threaded" and `webworkers` positioning. This is a genuine differentiator versus most JS frameworks, but the agent API has changed shape across releases.

**Server-side rendering.** Yew supports SSR and hydration (`ServerRenderer`, the `hydration` feature)[^4]. It is functional but younger and less turnkey than SSR in the JS ecosystem; you assemble the server (e.g. with Axum) and the hydration wiring yourself.

## Production Notes

**No 1.0 — expect breaking changes.** Yew is still on 0.x. Minor version bumps (0.18 → 0.19 → 0.20 → 0.21) have carried non-trivial API changes; the 0.19 function-component pivot in particular reshaped idiomatic code. Pin your version and read the migration guides before upgrading. Third-party component libraries frequently lag the latest Yew release.

**Bundle size.** A Rust/Wasm app carries the cost of shipping a `.wasm` binary. Even a small Yew app is typically hundreds of kilobytes before compression; `wasm-opt` (bundled with Trunk in release mode), aggressive `opt-level = "z"`/`"s"`, `lto`, and stripping debug info are standard mitigations. This is heavier than an equivalent Svelte or Solid bundle and is the main reason Yew is a poor fit for size-sensitive marketing pages.

**Compile times.** You inherit Rust's compile-time cost plus procedural-macro expansion of every `html!` block. Iteration is slower than a JS/TS hot-reload loop; `trunk serve` reloads are measured in seconds, not milliseconds. `cargo check` and splitting large components help.

**Rendering performance.** In cross-framework Wasm benchmarks Yew's virtual-DOM approach generally trails signal-based Rust frameworks (Leptos, Sycamore) on update-heavy workloads. For most CRUD UIs the difference is irrelevant; for large, frequently-updating tables/lists it is measurable, and there is no way to opt out of the diff model within Yew.

**Debugging.** Panics surface as a Wasm trap; enable `console_error_panic_hook` to get readable stack traces in the browser console. Source maps for Rust/Wasm are limited, so debugging often happens at the `web-sys` boundary.

**JS interop.** Reaching NPM packages or existing JS is possible but manual: you write `wasm-bindgen` `extern "C"` bindings or use `gloo`/`js-sys`. There is no automatic typing of arbitrary JS libraries.

## When to Use / When Not

**Use when:**
- You want to write a browser SPA entirely in Rust and share types/logic with a Rust backend.
- Your app benefits from moving CPU-heavy work into Web Workers (agents) while keeping one language.
- You value compile-time-checked markup and Rust's type system over JS iteration speed.
- The team already knows Rust and treats the front end as an extension of the same codebase.

**Avoid when:**
- Bundle size or first-paint latency is critical (a content site, landing page, or low-end-mobile target).
- You need the fastest possible Rust/Wasm reactivity — Leptos/Sycamore's signal model outperforms Yew's VDOM.
- You need a mature, stable, 1.0 API with rare breaking changes.
- The team is JS/TS-first and the Rust learning curve plus slower iteration outweigh the single-language benefit.

## Alternatives

- leptos-rs/leptos — use instead when you want fine-grained signal reactivity, smaller bundles, and first-class SSR/streaming in Rust.
- DioxusLabs/dioxus — use instead when you want one Rust UI codebase that also targets desktop and mobile, not just the browser.
- sycamore-rs/sycamore — use instead when you want a lightweight Solid-style signals framework with no virtual DOM.
- facebook/react — use instead when the team is JavaScript-first and you want the largest ecosystem and hiring pool.
- sveltejs/svelte — use instead when minimal bundle size and no framework runtime matter more than staying in Rust.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017–2018 | Initial releases; Elm-inspired struct components with `Message`/`update`. |
| 0.17 | 2020-08 | Consolidated component/services API. |
| 0.18 | 2021-03 | Final release before the function-component era. |
| 0.19 | 2021-11 | Major rewrite: function components + hooks, new `html!` internals[^2]. |
| 0.20 | 2023-01 | Function-component-first API, `use_prepared_state`, SSR improvements. |
| 0.21 | 2023-08 | Further hook/SSR refinements; current 0.x line. |

## References

[^1]: Yew documentation — "About Yew". https://yew.rs/docs/getting-started/introduction
[^2]: Yew blog, "Yew 0.19.0" — function components and hooks. https://yew.rs/blog/release-0-19-0
[^3]: Trunk — Wasm web application bundler. https://trunkrs.dev/
[^4]: Yew documentation — "Server-Side Rendering". https://yew.rs/docs/advanced-topics/server-side-rendering

## Tags

rust, webassembly, wasm, frontend, web-framework, spa, virtual-dom, component-model, jsx, wasm-bindgen, client-side-rendering, trunk
