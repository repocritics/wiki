# wasm-bindgen/wasm-bindgen

> The Rust ↔ JavaScript glue layer: proc-macros plus a post-processing CLI that let Wasm modules pass strings, structs, closures, and DOM handles across a boundary that natively only carries numbers.

[GitHub repo](https://github.com/wasm-bindgen/wasm-bindgen) ·
[Official guide](https://wasm-bindgen.github.io/wasm-bindgen/) ·
[License: Apache-2.0 OR MIT](https://github.com/wasm-bindgen/wasm-bindgen/blob/main/LICENSE-APACHE)

## Overview

`wasm-bindgen` solves a narrow but foundational problem: the WebAssembly ABI can only pass integers and floats across the JS↔Wasm boundary, yet real programs need to move strings, byte arrays, structs, closures, and references to live JavaScript objects. It closes that gap with a two-part design — a `#[wasm_bindgen]` procedural macro that annotates Rust imports/exports, and a CLI (`wasm-bindgen-cli`) that post-processes the compiler's raw `.wasm` output to emit the JavaScript glue and a rewritten `.wasm`. It began around 2017–2018 under the Rust and WebAssembly Working Group, largely driven by Alex Crichton, and has remained the canonical way to target the browser from Rust ever since[^1].

It is rarely used directly by application authors. Instead it sits at the bottom of nearly the entire Rust-on-the-web stack: `wasm-pack`, Trunk, and frameworks like Yew, Leptos, and Dioxus all generate `wasm-bindgen` calls under the hood. Its sibling crates — `js-sys` (raw bindings to the ECMAScript standard library), `web-sys` (DOM/Web API bindings auto-generated from WebIDL), and `wasm-bindgen-futures` (bridges Rust `Future` and JS `Promise`) — live in the same repository and version in lockstep.

The defining tension is that `wasm-bindgen` is glue, not a boundary you get for free. It has been on the `0.2.x` line for its entire existence[^2] and has never shipped a `1.0`, because the underlying WebAssembly proposals it was designed to eventually lean on — Interface Types, then Web IDL bindings, now the Component Model — have moved slowly. The long-promised future where JS shims disappear and Wasm calls DOM methods directly has not arrived, so the generated glue is still very much present in every build.

## Getting Started

The crate is used from Rust; the CLI must be installed separately and its version must match the crate version exactly:

```bash
cargo add wasm-bindgen
cargo install wasm-bindgen-cli   # OR: cargo binstall wasm-bindgen-cli
rustup target add wasm32-unknown-unknown
```

```rust
use wasm_bindgen::prelude::*;

// Import window.alert from JS into Rust.
#[wasm_bindgen]
extern "C" {
    fn alert(s: &str);
}

// Export greet from Rust to JS.
#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {name}!"));
}
```

```bash
cargo build --target wasm32-unknown-unknown --release
wasm-bindgen target/wasm32-unknown-unknown/release/hello.wasm \
  --out-dir ./pkg --target web
```

```js
import init, { greet } from "./pkg/hello.js";
await init();          // fetch + instantiate the .wasm
greet("World");
```

In practice most projects delegate the build+bindgen+bundling dance to `wasm-pack` or Trunk rather than invoking the CLI by hand.

## Architecture / How It Works

The pipeline has two compile-time stages and one runtime shim:

1. **Macro expansion.** `#[wasm_bindgen]` is a proc-macro that, for each annotated item, emits the real function plus a hidden "describe" function encoding the argument/return types. This type metadata is smuggled into a custom section of the `.wasm` so the CLI can recover the Rust-side type information the raw Wasm ABI has erased.
2. **CLI post-processing.** `wasm-bindgen-cli` reads that section, deletes it, rewrites the module, and generates a JavaScript file that marshals values across the boundary. Strings and byte slices are copied through the Wasm linear memory (pointer + length). Live JS objects are stored in a JS-side heap/table and referenced from Rust by index; `JsValue` is that index. Rust closures handed to JS become entries in a function table with manual lifetime management.
3. **Runtime glue.** The emitted JS wraps every call in encode/decode logic. `TextEncoder`/`TextDecoder` handle strings; a small allocator dance handles ownership of copied buffers.

`web-sys` and `js-sys` are not hand-written — `web-sys` is generated from browser WebIDL, producing tens of thousands of feature-gated bindings so you compile only the APIs you enable. `wasm-bindgen-futures` maps a Rust `Future` onto the microtask queue and converts to/from `Promise`, which is what makes `async fn` usable in the browser.

The `--target` flag of the CLI selects the JS output shape: `web` (native ES modules with an explicit `init`), `bundler` (for webpack/Vite, the default expectation of `wasm-pack`), `nodejs` (CommonJS), and `no-modules` (a global script). These are not interchangeable after the fact — the wrong target is a common first-run failure.

## Production Notes

**The CLI/crate version lock is the single most common footgun.** The `wasm-bindgen` crate version in your `Cargo.lock` and the installed `wasm-bindgen-cli` version must match exactly. A mismatch produces the notorious "schema version mismatch" error. CI must pin the CLI (or use `wasm-pack`, which vendors a compatible copy), and Dependabot bumps to the crate silently break local machines whose CLI wasn't reinstalled.

**`web-sys` compile times.** Because `web-sys` is a giant generated crate, enabling many features materially increases build time and can slow rust-analyzer. Enable only the specific features you use; do not blanket-enable everything for convenience.

**Boundary cost is real.** Every string or buffer crossing the boundary is a copy through linear memory, and every JS-object reference is a table indirection. Chatty designs that cross the boundary in tight loops are slow; batch data and cross once. The "even-faster-than-JavaScript DOM access" the README aspires to depends on Wasm proposals that are not yet the shipped default.

**Closures leak by default.** A `Closure` passed to JS must be kept alive on the Rust side for as long as JS can call it. `Closure::forget()` leaks it deliberately (fine for lifetime-of-page handlers); storing it in a struct is the correct pattern for temporary ones. Getting this wrong yields "closure invoked after being dropped" panics.

**Threads are not free.** Wasm threads require `SharedArrayBuffer`, atomics, a nightly toolchain flag, and cross-origin isolation (COOP/COEP headers) on the server. Most projects run single-threaded.

**`getrandom` and friends.** Crates that need entropy (via `getrandom`) require the `js` feature on `wasm32-unknown-unknown`, or they fail to link. This surprises people pulling in `uuid`, `rand`, or anything transitively depending on `getrandom`.

**No stability guarantee from the 0.2 line.** Being pre-1.0, minor `0.2.x` bumps can and do change generated output and MSRV. The MSRV policy is split: libraries target roughly the last two years of Rust (1.77 as of 0.2.118), while the CLI tracks a newer floor (1.86)[^3].

## When to Use / When Not

**Use when:**
- You are running Rust in a browser or JS runtime and need to exchange anything richer than numbers.
- You depend on `web-sys`/`js-sys` for DOM or Web API access, or on a framework (Yew, Leptos, Dioxus) that requires it.
- You want the mature, de facto standard path with the widest ecosystem support.

**Avoid when:**
- You target the WASI / WebAssembly Component Model outside the browser — use component tooling instead.
- Your workload is pure compute with a trivial numeric interface; raw `wasm32` exports with hand-written glue avoid the toolchain and version-lock entirely.
- Your source is C/C++ rather than Rust; Emscripten is the equivalent for that ecosystem.

## Alternatives

- rustwasm/wasm-pack — build orchestrator that wraps `wasm-bindgen` and produces npm-publishable packages; complementary, use it when you want a packaging pipeline rather than the raw CLI.
- bytecodealliance/wit-bindgen — bindings generator for the WebAssembly Component Model / WIT; use when targeting WASI components outside the browser instead of JS interop.
- emscripten-core/emscripten — C/C++ to Wasm plus JS glue; use when your source language is C/C++.
- koute/stdweb — the pre-`wasm-bindgen` Rust web-interop crate; unmaintained, historical interest only.
- rustwasm/gloo — higher-level idiomatic wrappers built on top of `web-sys`; use alongside, not instead of, `wasm-bindgen` when raw web-sys is too low-level.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2017-12-18 | Rust/WASM Working Group project begins[^1]. |
| 0.2.x line | 2018 | Settles on the `0.2` series; `js-sys`, `web-sys`, futures crates added over 2018–2019. |
| 0.2.93 | 2024-08-13 | Library MSRV 1.57, CLI MSRV 1.76[^3]. |
| 0.2.103 | 2025-09-17 | CLI MSRV raised to 1.82. |
| 0.2.106 | 2025-11-27 | Library MSRV 1.71. |
| 0.2.118 | 2026-04-10 | Library MSRV 1.77, CLI MSRV 1.86 — current 2-year policy split[^3]. |

## References

[^1]: The Rust and WebAssembly Working Group. https://rustwasm.github.io/ — repository created 2017-12-18 (GitHub API).
[^2]: `wasm-bindgen` on crates.io — the crate has published only `0.2.x` releases. https://crates.io/crates/wasm-bindgen
[^3]: MSRV Policy and MSRV History, project README. https://github.com/wasm-bindgen/wasm-bindgen#msrv-policy

## Tags

rust, webassembly, wasm, javascript, ffi, binding-generator, browser, dom, proc-macro, web-sys, interop
