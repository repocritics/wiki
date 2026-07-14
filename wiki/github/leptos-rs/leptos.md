# leptos-rs/leptos

> A full-stack, isomorphic Rust web framework built on fine-grained reactivity — no virtual DOM.

[GitHub repo](https://github.com/leptos-rs/leptos) ·
[Official website](https://leptos.dev) ·
[License: MIT](https://github.com/leptos-rs/leptos/blob/main/LICENSE-MIT)

## Overview

Leptos is a Rust framework for building web UIs, created by Greg Johnston (@gbj) with a first public release in 2022[^1]. It compiles to WebAssembly for the client and to native code for the server, letting one Rust codebase render on both sides. Its defining design choice is **fine-grained reactivity** borrowed from SolidJS: components run once to build the DOM, and a graph of signals, memos, and effects updates individual DOM nodes directly when data changes. There is no virtual DOM and no component re-render on state change.

The framework's central bet is that Rust's type system plus a reactive runtime can deliver both correctness and update performance without the diffing overhead of VDOM frameworks. In practice this places Leptos in the same design lineage as SolidJS and Sycamore, and against the VDOM approach of Yew and Dioxus. The tradeoff is a mental model that is unusual for developers coming from React: closures capture `Copy + 'static` signals that live in a reactive-ownership arena, and getting the ownership model wrong produces disposed-signal warnings rather than compile errors.

Leptos targets full-stack apps: client-side rendering (CSR), server-side rendering (SSR) with hydration, HTTP streaming of HTML (`<Suspense/>`), and isomorphic **server functions** — functions annotated `#[server]` that you call as if local but that execute only on the server, removing the need to hand-write a REST layer[^2]. It integrates with Axum and Actix on the server side.

## Getting Started

The build tool is `cargo-leptos`, which coordinates the wasm and server builds:

```bash
cargo install cargo-leptos --locked
cargo leptos new --git https://github.com/leptos-rs/start-axum
cd your-project
cargo leptos watch   # http://localhost:3000
```

A minimal reactive component (0.7 API):

```rust
use leptos::prelude::*;

#[component]
pub fn Counter(initial: i32) -> impl IntoView {
    let (value, set_value) = signal(initial);

    view! {
        <button on:click=move |_| set_value.update(|v| *v -= 1)>"-1"</button>
        <span>"Value: " {value} "!"</span>
        <button on:click=move |_| set_value.update(|v| *v += 1)>"+1"</button>
    }
}

fn main() {
    leptos::mount::mount_to_body(|| view! { <Counter initial=3 /> });
}
```

Client-only apps can also be built with Trunk instead of `cargo-leptos`.

## Architecture / How It Works

The system is built from a small set of reactive primitives:

- **Signals** (`signal()`, `RwSignal`) — the reactive state cells. Since 0.7 they are `Copy + 'static` handles into an arena, so they move freely into closures without cloning or `Rc`. This ergonomic win is Leptos's signature innovation, but it means a signal is a key, not the value — accessing one after its owning scope is disposed is a runtime condition, not a compile error.
- **Memos** — cached derived values that recompute only when dependencies change.
- **Effects** — side effects (DOM updates, logging) that re-run when their tracked signals change. Effects run only on the client.
- **`view!` macro** — a JSX-like DSL that expands to the builder API (`div().child(...)`). Because updates are fine-grained, `{value}` in a view compiles to code that patches exactly one text node.

The 0.7 release was a large internal rewrite. The renderer was rebuilt (the `tachys` crate) and the reactive core was split into a standalone `reactive_graph` crate, moving away from the older single-runtime, generational-arena model of 0.5/0.6[^3]. A planned "generic rendering" abstraction — one view layer targeting DOM and native GUI — was attempted during 0.7 but shelved because the volume of generics overwhelmed the Rust compiler on larger apps[^4].

**Server functions** (`server_fn` crate) serialize their arguments and return values (serde), generate a POST endpoint on the server, and generate a matching client-side call. **Resources** are async reactive values that integrate with `<Suspense/>` and `<Await/>` for streaming. An experimental **islands** mode ships mostly-static HTML and hydrates only interactive regions to shrink the wasm payload.

## Production Notes

**Reactive ownership is the main footgun.** Signals are owned by the reactive graph node that created them and are disposed when that node is. Storing a signal past its owner's lifetime, or reading it in a detached context, yields "attempted to access a reactive value after disposal" warnings rather than a type error. `StoredValue` / `store_value` and explicit `Owner` handling are the escape hatches. This is the single biggest source of confusion for new users and the reason the repo maintains a dedicated `COMMON_BUGS.md`[^5].

**Hydration mismatches.** SSR emits HTML that the client wasm must hydrate against. If server and client render different trees (non-deterministic data, time, conditional-on-`cfg` markup), hydration produces broken event handlers or panics. Errors are often opaque; the fix is ensuring identical render inputs on both sides.

**WASM binary size.** Rust-to-wasm output is large by default. Production builds need `opt-level = "z"` or `"s"`, LTO, `wasm-opt` (via `cargo-leptos`), and ideally islands mode to reduce shipped JS/wasm. Expect meaningful tuning effort versus a JS framework's baseline.

**Randomness on wasm.** `rand` / `getrandom` need their JS backend enabled explicitly (`getrandom = { version = "0.2", features = ["js"] }`) or wasm builds fail or produce no entropy. Leptos configures this for itself but not for your direct dependencies[^6].

**Compile times & tooling.** Standard Rust build-time costs apply, amplified by heavy proc-macros (`view!`, `#[server]`). `cargo leptos watch` rebuilds both targets. `view!` macro errors can be cryptic when a type doesn't implement `IntoView`.

**Version migrations.** APIs are described by the maintainer as broadly settled, but 0.5 (reactivity overhaul), 0.6, and especially 0.7 (renderer + `create_signal`→`signal`, `leptos::*`→`leptos::prelude::*`) each required real code changes. Pin the framework and its integration crates (`leptos_axum` / `leptos_actix`) to matching minor versions.

## When to Use / When Not

**Use when:**
- You want a full Rust stack (client + server + shared types) with no separate API layer, via server functions.
- You need fine-grained update performance and want to avoid VDOM diffing overhead.
- Your team is already comfortable with Rust and wants end-to-end type safety across the network boundary.
- You want SSR, streaming, and hydration in one framework rather than assembling them.

**Avoid when:**
- You need rapid prototyping or a large hiring pool — the ecosystem and mental model are niche versus React.
- You're building a primarily desktop or cross-platform native app — Dioxus is a better fit there.
- You can't budget wasm size tuning or the reactive-ownership learning curve.
- You depend on a mature third-party UI component ecosystem — Leptos's is young.

## Alternatives

- yewstack/yew — the most-established Rust web UI framework; VDOM + component re-render model, larger ecosystem. Use when you want the safer, more documented Rust option and don't need fine-grained updates.
- dioxuslabs/dioxus — VDOM, component-scoped reactivity, first-class desktop/mobile/native. Use when cross-platform native (not just web) is the priority.
- sycamore-rs/sycamore — the closest peer: also fine-grained and SolidJS-inspired. Use when you prefer its templating and it fits; Leptos currently has the larger community and faster development.
- solidjs/solid — the JavaScript framework Leptos's reactivity is modeled on. Use when you want the same programming model without adopting Rust and wasm.
- rust-lang/rust — the language itself; the borrow checker and wasm target underpin everything Leptos does.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.x / 0.1 | 2022-10 | Initial public release; fine-grained reactive core[^1]. |
| 0.5 | 2023-09 | Major reactive-system overhaul (unified graph). |
| 0.6 | 2024-02 | Refinements; Actix/Axum integration maturing. |
| 0.7 | 2024-12 | Renderer rewrite (`tachys`), `reactive_graph` crate split, `signal()` API, `leptos::prelude`[^3]. |

Dates for 0.5/0.6 are approximate release windows; see the changelog for exact tags.

## References

[^1]: Leptos repository and website. https://leptos.dev
[^2]: Server functions documentation. https://docs.rs/server_fn/latest/server_fn/
[^3]: Leptos book and API docs (0.7 series). https://leptos-rs.github.io/leptos/ · https://docs.rs/leptos/latest/leptos/
[^4]: README FAQ, "Can I use this for native GUI?" — on the shelved generic-rendering effort. https://github.com/leptos-rs/leptos#can-i-use-this-for-native-gui
[^5]: Common bugs guide. https://github.com/leptos-rs/leptos/blob/main/docs/COMMON_BUGS.md
[^6]: README, "Random numbers on wasm (`rand` / `getrandom`)". https://github.com/leptos-rs/leptos

## Tags

rust, webassembly, wasm, web-framework, fine-grained-reactivity, ssr, hydration, isomorphic, full-stack, signals, frontend
