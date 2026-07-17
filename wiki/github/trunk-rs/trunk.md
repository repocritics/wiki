# trunk-rs/trunk

> A zero-to-low-config bundler and dev server for client-side Rust WASM web apps, driven by a source HTML file.

[GitHub repo](https://github.com/trunk-rs/trunk) ·
[Official website](https://trunk-rs.github.io/trunk/) ·
[License: MIT OR Apache-2.0](https://github.com/trunk-rs/trunk#license)

## Overview

Trunk builds, bundles, and serves Rust applications that compile to WebAssembly for the browser. It occupies the niche that webpack/Vite fill in the JavaScript world: it takes a `wasm32-unknown-unknown` Rust crate plus assets (CSS, SCSS, images, JS snippets) and produces a deployable `dist/` directory, with a watch-mode dev server and live reload on top. It is the de facto build tool for the client-side Rust framework ecosystem — Yew, Leptos (CSR mode), Dioxus, Sycamore, and Seed all document Trunk as a supported entry point.

The defining design choice is that the **source HTML file is the manifest**. Rather than a JS-style config graph, Trunk scans `index.html` for `<link data-trunk .../>` directives and derives the whole build from them. This keeps trivial apps genuinely config-free, but means the mental model is inverted from Cargo-centric tooling: the HTML, not `Cargo.toml`, is the build's entry point. Originally authored by Anthony Dodd (`thedodd/trunk`), the project later moved to the community-run `trunk-rs` organization.[^1]

The project has been in `0.x` for its entire life. At ~4,350 stars it is well-established within Rust-WASM but small next to mainstream JS bundlers, and its release cadence has slowed: the last stable line is `v0.21.x` (May 2025), with `v0.22.0-beta.1` landing March 2026 — a long gap that signals maintenance-mode velocity rather than active feature push.[^2]

## Getting Started

```bash
# Install a precompiled binary (fastest)
cargo binstall trunk
# or compile from source
cargo install trunk --locked
rustup target add wasm32-unknown-unknown
```

```html
<!-- index.html — the build manifest -->
<!DOCTYPE html>
<html>
  <head>
    <link data-trunk rel="rust" data-wasm-opt="z" />
    <link data-trunk rel="scss" href="styles/main.scss" />
    <link data-trunk rel="icon" href="favicon.png" />
  </head>
  <body></body>
</html>
```

```bash
trunk serve         # dev server + watch + live reload on :8080
trunk build --release   # optimized output to dist/
```

## Architecture / How It Works

Trunk's pipeline runs in stages, all keyed off `index.html`:

1. **Parse HTML** for `data-trunk` link directives, each mapping to an asset pipeline (`rust`, `sass`/`scss`, `css`, `tailwind`, `icon`, `copy-file`, `copy-dir`, `inline`, `js`).
2. **Rust pipeline** invokes `cargo build --target wasm32-unknown-unknown`, then runs **`wasm-bindgen`** to generate the JS glue and the `.wasm` binding, and optionally **`wasm-opt`** (Binaryen) to shrink the output.
3. **Asset pipelines** compile SCSS (via a bundled Rust Sass implementation), run Tailwind, hash filenames, and emit into `dist/`.
4. **Rewrite** `index.html` to reference the hashed, fingerprinted outputs.
5. **Serve** (`trunk serve`) adds a file watcher, incremental rebuild, and a WebSocket-based live-reload channel, plus optional backend proxies.

A notable and load-bearing behavior: Trunk **auto-downloads its own tool dependencies** — `wasm-bindgen-cli`, `wasm-opt`, `sass`, `tailwindcss` — into a local cache the first time they are needed, rather than requiring them on `PATH`. The `wasm-bindgen-cli` version is chosen to match the `wasm-bindgen` crate version resolved in your `Cargo.lock`; a mismatch there is the single most common source of build failures.[^3]

Configuration beyond the HTML is optional and lives in `Trunk.toml`: `[build]`, `[serve]`, `[tools]` (pin tool versions), `[[proxy]]` (backend/API proxying in dev), and `[[hooks]]` (shell hooks at build stages). Deploying under a non-root path requires setting `--public-url` / `public_url` so the emitted asset URLs and the reload WebSocket resolve correctly.

Trunk is **client-side only**. It produces a static SPA bundle and has no server-side rendering. This is the ecosystem's central tension: Leptos and Dioxus each ship their *own* fullstack CLIs (`cargo-leptos`, `dx`) for SSR/hydration, and once you need SSR you generally leave Trunk behind. Trunk owns the CSR/static-SPA lane cleanly and stops at its boundary.

## Production Notes

- **`0.x` means breaking minors.** Config format and directive behavior have shifted across minor versions (e.g. the `0.16 → 0.17 → 0.18` era changed defaults and options). Pin an exact version in CI; do not float.
- **Build-time network access.** The tool auto-download behavior fails in air-gapped or restricted CI runners. Pre-seed the tool cache, pin versions in `[tools]`, or pre-install the binaries and point Trunk at them; budget setup time for locked-down environments.
- **wasm-bindgen version drift.** If the `wasm-bindgen` crate and the CLI Trunk uses diverge, builds break with opaque errors. Keep the crate pinned and let Trunk resolve the matching CLI, or pin both.
- **Binary size is your problem.** Rust-WASM outputs are large by default. `--release` plus `wasm-opt` (`data-wasm-opt="z"`/`"s"`) helps, but expect hundreds of KB to multiple MB; profile with `twiggy` and trim panic/formatting machinery. Trunk does not shrink code it cannot see.
- **`wasm-opt` cost.** Optimization is slow and memory-hungry on large binaries; it is skipped by default in debug builds and worth disabling explicitly for fast dev loops.
- **Live reload behind proxies/TLS.** The reload WebSocket needs correct `--public-url` and, behind a reverse proxy or HTTPS terminator, explicit websocket/host configuration or it silently fails to reconnect.
- **No SSR / SEO.** Being CSR-only, initial HTML is an empty shell; content-indexing and first-paint SEO require prerendering or a different tool.

## When to Use / When Not

**Use when:**
- You are building a client-side Rust WASM SPA (Yew, Leptos CSR, Sycamore, Dioxus, Seed) and want a working dev server + live reload with near-zero config.
- You want an integrated asset pipeline (SCSS, Tailwind, image hashing) without wiring up a JS bundler alongside Rust.
- You are deploying a static bundle to any host (Netlify, Pages, S3, nginx) and want fingerprinted output.

**Avoid when:**
- You need server-side rendering or fullstack hydration — use the framework's own CLI (`cargo-leptos`, Dioxus `dx`).
- You are shipping a WASM *library* to consume from JavaScript/npm — `wasm-pack` targets that directly.
- You require a stable, `1.0`-guaranteed tool with a strict semver contract, or you must build fully offline without setup work.

## Alternatives

- rustwasm/wasm-pack — packages Rust WASM as an npm-consumable module with no HTML pipeline or dev server; use it when the artifact is a library for a JS build, not an app.
- rustwasm/wasm-bindgen — the binding generator Trunk itself drives; use directly when you need a bespoke build pipeline and don't want Trunk's HTML-first model.
- DioxusLabs/dioxus — the `dx` CLI builds, serves, and hot-reloads Dioxus apps including fullstack/SSR; use it when you're on Dioxus and need more than CSR.
- leptos-rs/leptos — pairs with cargo-leptos for Leptos server-side rendering; use when you need Leptos SSR rather than Trunk's static output.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-08 | Created as `thedodd/trunk`; HTML-as-manifest WASM bundler.[^1] |
| v0.16.0 | 2022-06-30 | Later `0.16`-era line under the `trunk-rs` org. |
| v0.17.0 | 2023-06-27 | Config/directive changes; asset pipeline refinements. |
| v0.18.0 | 2023-12-12 | Minor line, continued pipeline/tooling work. |
| v0.19.0 | 2024-03-08 | Feature and fix line. |
| v0.20.0 | 2024-05-03 | Feature and fix line. |
| v0.21.0 | 2024-10-14 | Current stable major line.[^2] |
| v0.21.14 | 2025-05-08 | Latest stable patch. |
| v0.22.0-beta.1 | 2026-03-10 | Next line in beta after a long gap. |

## References

[^1]: Trunk repository and project home, `trunk-rs` organization (originally `thedodd/trunk`). https://github.com/trunk-rs/trunk
[^2]: Trunk releases — version tags and dates verified via the GitHub Releases API; latest stable `v0.21.14` (2025-05-08), `v0.22.0-beta.1` (2026-03-10). https://github.com/trunk-rs/trunk/releases
[^3]: Trunk documentation — install, assets, configuration, and tool management. https://trunk-rs.github.io/trunk/

## Tags

rust, wasm, webassembly, bundler, build-tool, dev-server, frontend, spa, wasm-bindgen, asset-pipeline, cli
