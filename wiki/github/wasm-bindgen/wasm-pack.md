# wasm-bindgen/wasm-pack

> A CLI that wraps the Rust → WebAssembly toolchain into one command and emits an npm-publishable package.

[GitHub repo](https://github.com/wasm-bindgen/wasm-pack) ·
[Official website](https://wasm-bindgen.github.io/wasm-pack/) ·
[License: Apache-2.0](https://github.com/wasm-bindgen/wasm-pack/blob/master/LICENSE-APACHE) (dual-licensed MIT OR Apache-2.0, the Rust ecosystem norm)

## Overview

wasm-pack is a build-orchestration CLI, not a library or a compiler. It sequences the several tools you would otherwise invoke by hand to turn a Rust crate into a WebAssembly module that JavaScript can call: `cargo build` against the `wasm32-unknown-unknown` target, `wasm-bindgen` to generate the JS/TypeScript glue, optionally `wasm-opt` (from Binaryen) to shrink the binary, and finally a generated `package.json` so the result can be `npm publish`ed or consumed by a bundler[^1]. It was started by Ashley Williams (ashleygwilliams) in 2018 under the Rust WebAssembly working group and is now maintained principally by drager[^2].

The tool's defining tension is that it is a thin, opinionated wrapper over `wasm-bindgen`, which does the actual work. That is the point — it hides a fiddly multi-step pipeline behind `wasm-pack build` — but it also means wasm-pack inherits `wasm-bindgen`'s capabilities and constraints while adding its own opinions about output layout, target presets, and tool version management. Teams that outgrow those opinions frequently drop wasm-pack and call `cargo` + `wasm-bindgen-cli` directly.

The project has moved organizationally: the repo now lives under the `wasm-bindgen` GitHub org (redirected from the former `rustwasm/wasm-pack`) after the Rust WebAssembly working group's repositories were consolidated. Release cadence has slowed to roughly one or two releases per year; it is stable and widely used but no longer under heavy feature development, and much of the ecosystem's momentum for full web *apps* has shifted to Trunk and, for the Component Model, to cargo-component.

## Getting Started

```bash
# Install (requires a Rust toolchain; wasm-pack fetches the wasm32 target itself)
cargo install wasm-pack
# or the prebuilt installer:
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
```

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
```

```bash
# Build an npm package into ./pkg for a bundler (webpack/rollup/vite)
wasm-pack build --target bundler
# or ES modules usable directly in the browser:
wasm-pack build --target web --release
```

## Architecture / How It Works

`wasm-pack build` is a pipeline coordinator. Each stage shells out to a separate tool:

1. **Toolchain checks** — confirms a Rust toolchain and installs the `wasm32-unknown-unknown` target if missing.
2. **Compile** — runs `cargo build --target wasm32-unknown-unknown` (release or dev profile). The crate must be `crate-type = ["cdylib"]` for a linkable wasm module.
3. **Bindgen** — downloads/uses a `wasm-bindgen-cli` whose version is pinned to the `wasm-bindgen` dependency in your `Cargo.lock`, then runs it to emit the `.wasm`, the JS shim, and `.d.ts` TypeScript types.
4. **Optimize** — if enabled, downloads and runs `wasm-opt` (Binaryen) to reduce code size.
5. **Package** — writes a `package.json` into `pkg/` describing the module for npm.

**Target presets** are the main abstraction wasm-pack adds on top of `wasm-bindgen`, which otherwise takes many flags. `--target bundler` (default) emits ESM that assumes a bundler will handle the `.wasm` import; `--target web` emits a module with an explicit async `init()` you call before use; `--target nodejs` emits CommonJS with synchronous instantiation; `--target no-modules` emits a global-script build; Deno is also supported. Picking the wrong target is the single most common first-time confusion, because the generated `import`/init contract differs across all of them.

**Testing.** `wasm-pack test` drives `wasm-bindgen-test`, running `#[wasm_bindgen_test]` functions either in Node or in a real headless browser via `chromedriver`/`geckodriver` (`--chrome`, `--firefox`, `--node`, `--headless`). This is one of the few genuinely load-bearing pieces beyond the build wrapper, since browser-hosted wasm tests are otherwise awkward to wire up.

**Version coupling** is the architecture's sharp edge: wasm-pack, the `wasm-bindgen` crate, the `wasm-bindgen-cli` binary, and (transitively) `wasm-opt` must all be mutually compatible. wasm-pack tries to select a matching `wasm-bindgen-cli`, but mismatches between the crate version and an externally installed CLI produce cryptic "schema version" errors.

## Production Notes

- **Network at build time.** wasm-pack downloads `wasm-bindgen-cli` and `wasm-opt` binaries on demand and caches them. In sandboxed or air-gapped CI this fails or hangs; the fix is to pre-install `wasm-bindgen-cli` at the exact matching version and/or use `WASM_PACK_CACHE` / pin tool versions. This is the most reported CI footgun.
- **`wasm-opt` toggling.** The optimizer occasionally miscompiles or chokes on newer wasm features (reference types, bulk memory) depending on the Binaryen version pulled. Disabling it via `[package.metadata.wasm-pack.profile.release] wasm-opt = false` in `Cargo.toml` is a common workaround when builds break after a toolchain bump.
- **Version-skew errors.** "rust wasm-bindgen vX.Y.Z but the CLI is vA.B.C" errors mean the crate and CLI disagree. Keep `wasm-bindgen` in `Cargo.toml` and any manually installed `wasm-bindgen-cli` in lockstep; prefer letting wasm-pack manage the CLI.
- **`getrandom` / randomness.** Crates that transitively use `getrandom` need its `js` feature enabled for the browser, or the wasm module panics at first RNG use. This surfaces constantly with `uuid`, `rand`, and crypto crates.
- **Output layout is opinionated.** The `pkg/` directory and generated `package.json` assume npm publishing. Projects that just want a `.wasm` + JS to drop into an existing app often find the packaging overhead more friction than help, and switch to raw `wasm-bindgen-cli`.
- **Not for whole web apps.** wasm-pack targets the "publish a wasm library to npm" use case. For a Rust frontend app (Yew, Leptos) with asset pipelining and a dev server, Trunk is the better-fit tool and wasm-pack is the wrong layer.
- **Maintenance pace.** Because releases are infrequent, support for brand-new wasm proposals or Rust target changes can lag; teams on the bleeding edge sometimes vendor `wasm-bindgen-cli` directly rather than wait for a wasm-pack release.

## When to Use / When Not

**Use when:**
- You are shipping a Rust-authored WebAssembly *library* to npm or a JS bundler and want one command instead of a hand-rolled `cargo` + `wasm-bindgen` + `wasm-opt` script.
- You want `wasm-bindgen-test` browser/Node testing set up for you.
- You value a stable, well-trodden path over the newest features.

**Avoid when:**
- You are building a full Rust web application (use Trunk).
- You are targeting the WebAssembly Component Model / WASI rather than JS interop (use cargo-component).
- You need tight control over each build step or bleeding-edge wasm features (call `cargo` + `wasm-bindgen-cli` directly).
- Your CI cannot fetch binaries at build time and you are unwilling to pin/pre-install the toolchain.

## Alternatives

- rustwasm/wasm-bindgen — the binding generator wasm-pack wraps; use it directly when you want full control of the build/output pipeline.
- trunkrs/trunk — use instead when building a whole Rust→wasm frontend app (Yew/Leptos) with a dev server and asset bundling, not a publishable package.
- bytecodealliance/cargo-component — use instead when targeting the WebAssembly Component Model / WASI rather than JavaScript interop.
- emscripten-core/emscripten — use instead when compiling C/C++ (not Rust) to WebAssembly.
- stdweb (unmaintained) — historical predecessor for Rust-in-browser; use wasm-bindgen instead, stdweb is dead.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-02 | Initial release; announced by ashleygwilliams under the Rust WASM working group[^2]. |
| 0.8.x | 2019 | Target presets and `wasm-pack test` maturation. |
| 0.9.0 | 2019 | Stabilized build/test workflow. |
| 0.10.0 | 2021 | Toolchain and dependency updates. |
| 0.11.0 | 2023 | Maintenance release under drager's stewardship. |
| 0.12.x | 2023 | Continued maintenance; tooling compatibility fixes. |
| 0.13.x | 2024 | Recent line; ongoing compatibility with newer Rust/wasm-bindgen. |

## References

[^1]: wasm-pack documentation ("The `wasm-pack` book"). https://wasm-bindgen.github.io/wasm-pack/book/
[^2]: Repository governance section, README: project started by ashleygwilliams, maintained by drager; repo now under the `wasm-bindgen` org. https://github.com/wasm-bindgen/wasm-pack

## Tags

rust, webassembly, wasm, cli, build-tool, npm, wasm-bindgen, javascript-interop, toolchain, frontend
