# not-fl3/macroquad

> A raylib-inspired, immediate-mode 2D game library for Rust that compiles the same code to desktop, web, Android, and iOS.

[GitHub repo](https://github.com/not-fl3/macroquad) ·
[Docs (docs.rs)](https://docs.rs/macroquad) ·
[License: Apache-2.0](https://github.com/not-fl3/macroquad/blob/master/LICENSE-APACHE)

## Overview

macroquad is a small game library authored primarily by Fedor Logachev (not-fl3), first
published in 2020[^1]. Its explicit design goal is approachability: it copies raylib's
flat, function-call-per-primitive API (`draw_line`, `draw_rectangle`, `draw_text`) so that
a beginner can draw something on screen without touching an ECS, a scene graph, or a render
graph. As of 2026 it has ~4.5k GitHub stars and continues to receive commits (last push
mid-2026), but it is effectively a solo/small-maintainer project — a meaningful bus-factor
consideration for anyone adopting it for a long-lived product.

The defining architectural fact is that macroquad is a thin high-level layer over
**miniquad**, a separate crate by the same author that abstracts windowing and the graphics
backend (OpenGL 2.1 / GLES / WebGL / Metal)[^2]. miniquad is what makes the "same code
everywhere" claim true; macroquad adds batching, an asset loader, an immediate-mode UI, an
audio module, coroutines, and a global rendering context. The central tension is that this
convenience is bought with a **thread-local global context** and a single-threaded, immediate-mode
model — excellent for jams and small 2D games, awkward the moment you want threads,
multiple windows, or a large structured codebase.

## Getting Started

```sh
cargo init --bin
cargo add macroquad   # or: macroquad = "0.4" in Cargo.toml
cargo run
```

```rust
use macroquad::prelude::*;

#[macroquad::main("BasicShapes")]
async fn main() {
    loop {
        clear_background(RED);
        draw_line(40.0, 40.0, 100.0, 200.0, 15.0, BLUE);
        draw_rectangle(screen_width() / 2.0 - 60.0, 100.0, 120.0, 60.0, GREEN);
        draw_circle(screen_width() - 30.0, screen_height() - 30.0, 15.0, YELLOW);
        draw_text("IT WORKS!", 20.0, 20.0, 30.0, DARKGRAY);
        next_frame().await
    }
}
```

The `#[macroquad::main]` attribute macro generates the real `main`, initializes the miniquad
window plus the global rendering context, and drives your `async fn` as the game loop.

## Architecture / How It Works

**Two crates, one story.** miniquad owns the event loop, the window, input, and a minimal
graphics abstraction that targets whatever backend the platform provides. macroquad sits on
top and provides everything a raylib user expects: a 2D quad batcher, `Texture2D` / `Image`
loading, fonts and text, camera helpers, a small 3D primitive set (`draw_cube`, models), an
audio module, and an immediate-mode UI (`root_ui`). Almost all of macroquad's public API
reads and mutates a single hidden context.

**Why async.** The `.await` in every example is not an async runtime — there is no executor,
no `tokio`, no `futures-rs`[^3]. Rust futures are used purely as a portable way to suspend
and resume the main loop's stack. On web you cannot block the browser's event loop, and
`next_frame().await` is how macroquad yields control back to the host each frame without a
platform-specific loop. Asset loaders (`load_texture`, `load_string`) are `async` for the
same reason: on WASM they resolve via `fetch`, and awaiting is how the frame yields while
the file arrives.

**Rendering.** Draw calls accumulate geometry and are flushed in batches (a state change —
new texture, new blend mode, new render target — forces a flush). This automatic batching is
the main reason large numbers of simple sprites stay cheap without the caller managing draw
calls manually.

**Web packaging.** A WASM build produces a `.wasm` that is loaded by a JavaScript shim,
`mq_js_bundle.js`, which wires the canvas, input, and GL context. The bundle is versioned
alongside miniquad; mismatched shim/crate versions are a recurring source of runtime
breakage on the web target.

## Production Notes

The convenience layer has sharp edges that only show up past the prototype stage:

- **Global, single-threaded context.** State lives in a thread-local set up by the macro.
  Textures and GPU resources must be created on the main thread, and most macroquad calls
  are not usable from worker threads. Background threads can compute data, but hand it back
  to the main loop to draw. This also makes multiple independent game contexts effectively
  impossible.
- **The built-in UI is minimal.** `root_ui` (megaui) covers debug panels and simple menus
  but is not a serious application-UI toolkit and sees little active development. Most teams
  that need real UI reach for an egui integration (`egui-macroquad`) instead.
- **WASM shim coupling.** Keep `mq_js_bundle.js` in lockstep with the miniquad version your
  macroquad pulls in. A stale hosted bundle against a newer crate is a classic "works
  locally, blank canvas in the browser" failure.
- **Android/iOS builds are out-of-band.** Mobile packaging is not `cargo run`; Android has
  historically gone through a dedicated Docker/NDK image and iOS requires manual `.app`
  assembly and provisioning[^4]. Budget real time for first mobile deploy.
- **Old GL floor.** Targeting OpenGL 2.1-era features keeps portability wide but means no
  compute shaders and a conservative feature set; custom `material`/shader work is GLSL at a
  low version.
- **2D-first.** The 3D primitives exist but macroquad is not a 3D engine — no scene graph,
  culling, lighting, or asset pipeline. Reaching for it to build 3D fights the grain.
- **Bus factor.** With ~331 open issues and essentially one core maintainer, review and
  release cadence is irregular. Pin versions and expect to read source when debugging.

## When to Use / When Not

**Use when:**
- You are building a 2D game, tool, or visualization and want to draw immediately.
- You need one codebase to ship to desktop and the web (and optionally mobile) with minimal
  platform-specific code.
- You value fast compile times and a tiny dependency tree over engine features.
- You are doing a game jam, teaching, or prototyping.

**Avoid when:**
- You need a full engine: ECS, 3D pipeline, editor, asset importing — use Bevy or Fyrox.
- Your architecture depends on multithreaded rendering or multiple GPU contexts.
- You need a mature, richly-staffed project with predictable releases and SLAs.
- Application-grade UI is central to the product.

## Alternatives

- bevyengine/bevy — full data-driven ECS engine (2D and 3D); use when you want engine
  structure and are willing to accept larger builds and a steeper model.
- ggez — LÖVE-inspired Rust 2D framework; use when you prefer an explicit `Context`-passing
  API over macroquad's hidden global state.
- FyroxEngine/Fyrox — full 3D engine with a scene editor; use when the project is 3D.
- not-fl3/miniquad — macroquad's own lower layer; use when you want just windowing plus a
  minimal GL/Metal/WebGL abstraction and will build the rest yourself.
- raysan5/raylib — the C library macroquad imitates; use when you want C or bindings in
  another language.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2020 | Initial release; high-level layer over miniquad[^1]. |
| 0.3 | 2021 | Rendering-path rework; batching and texture handling reworked. |
| 0.4 | 2023 | Current series (0.4.x); ongoing patch releases. |

(Exact release dates beyond the year are omitted where not independently verified; see
crates.io / docs.rs for the precise version history.)

## References

[^1]: macroquad on crates.io — version history and metadata. https://crates.io/crates/macroquad
[^2]: miniquad — cross-platform windowing and graphics backend by the same author.
      https://github.com/not-fl3/miniquad
[^3]: macroquad README, "async/await" section — futures used only for cross-platform main
      loop, no runtime involved. https://github.com/not-fl3/macroquad
[^4]: macroquad iOS build article. https://macroquad.rs/articles/ios/

## Tags

rust, game-engine, game-development, 2d-graphics, immediate-mode, wasm, cross-platform, raylib-inspired, miniquad, gamedev
