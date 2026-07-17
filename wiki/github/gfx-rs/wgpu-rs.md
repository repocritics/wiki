# gfx-rs/wgpu-rs

> The original idiomatic Rust wrapper for the WebGPU API — archived in 2021 after being folded into the main `gfx-rs/wgpu` repository.

[GitHub repo](https://github.com/gfx-rs/wgpu-rs) ·
[Official website](https://wgpu.rs) ·
[License: MPL-2.0](https://github.com/gfx-rs/wgpu-rs/blob/master/LICENSE)

## Overview

`wgpu-rs` was the safe, idiomatic Rust front-end for the WebGPU API, built under the gfx-rs organization[^1]. It sat on top of `wgpu-core` (the pure-Rust implementation of the WebGPU specification) and exposed a hand-written Rust API that mirrored the in-progress W3C WebGPU standard rather than any single native graphics API. The goal was one portable API that compiles down to Vulkan, Metal, DirectX 12, and OpenGL/GLES on native targets, and to the browser's own WebGPU/WebGL implementation when built for WebAssembly.

**This repository is archived and no longer developed.** Its README is a single line pointing readers to the `wgpu` crate directory inside [gfx-rs/wgpu](https://github.com/gfx-rs/wgpu)[^2]. In mid-2021 the gfx-rs maintainers consolidated the previously separate `wgpu` (core), `wgpu-native` (C bindings), and `wgpu-rs` (Rust wrapper) repositories into a single monorepo. The Rust wrapper became the top-level `wgpu` crate published on crates.io. Anyone starting new work should depend on that crate, not this repo.

The page exists because `wgpu-rs` is still widely referenced in older tutorials, blog posts, and dependency graphs (the "Learn Wgpu" material and early Bevy versions were written against it), and because the last commit here (June 2021) is a useful historical snapshot of the API before the `wgpu-hal` rewrite landed in the successor repo.

## Getting Started

Do not start here. Use the current crate from the consolidated repository:

```bash
cargo add wgpu winit pollster
```

A minimal instance/adapter/device handshake — the shape below is essentially unchanged from the `wgpu-rs` era, though exact type and method names have drifted across versions:

```rust
let instance = wgpu::Instance::default();
let surface = instance.create_surface(&window)?;

let adapter = pollster::block_on(instance.request_adapter(
    &wgpu::RequestAdapterOptions {
        power_preference: wgpu::PowerPreference::HighPerformance,
        compatible_surface: Some(&surface),
        force_fallback_adapter: false,
    },
)).expect("no adapter");

let (device, queue) = pollster::block_on(adapter.request_device(
    &wgpu::DeviceDescriptor::default(),
    None,
))?;
```

If you genuinely need the archived crate for reproducing an old build, pin the exact `wgpu = "=0.x"` version the project used; the 0.x series was not API-stable across minor releases.

## Architecture / How It Works

At the time of archival, the stack was three layers, split across three repositories:

1. **`wgpu-rs`** (this repo) — the safe Rust API. Handles/IDs (`Device`, `Queue`, `Buffer`, `Texture`, `RenderPipeline`, `BindGroup`) are thin wrappers over integer identifiers managed by the core.
2. **`wgpu-core`** — the specification implementation: validation, resource tracking, command encoding, and the ID allocator. This is the layer that enforces WebGPU's safety and validation rules regardless of the front-end language.
3. **`gfx-hal`** — the hardware abstraction layer that `wgpu-core` used to reach native APIs (Vulkan, Metal, DX12, DX11, GL) via the `gfx-backend-*` crates.

The reliance on `gfx-hal` is the key historical detail. `gfx-hal` was a Vulkan-shaped portability layer for the whole gfx-rs project. Shortly after this repo was archived, the successor `gfx-rs/wgpu` replaced `gfx-hal` with a purpose-built **`wgpu-hal`**, a lighter abstraction designed specifically for WebGPU's needs. Code and internals described in `wgpu-rs`-era material therefore do not match the current crate below the public API surface.

Shader input was WGSL (the WebGPU Shading Language) plus SPIR-V and GLSL, translated by **naga**[^3], the gfx-rs shader translation crate that is still central to the successor. On the web, `wgpu-rs` compiled to WebAssembly and delegated to the browser's WebGPU implementation where available, falling back to a WebGL backend otherwise.

## Production Notes

- **It is archived. Treat any use as legacy maintenance, not a foundation.** No security fixes, no new backend support, no compatibility with current `naga` or `winit` versions land here.
- **Version churn was severe.** During the `wgpu-rs` period the WebGPU spec itself was unstable, so nearly every 0.x release renamed types, changed descriptor fields, or altered the surface/swapchain model. Upgrading between minors routinely required non-trivial rewrites — this is the single most-cited pain point in "Learn Wgpu"-era tutorials, which had to be revised repeatedly.
- **`winit` coupling.** Windowing was almost always done through `winit`, and `winit`'s own breaking changes (event-loop ownership, surface creation) compounded the `wgpu-rs` churn. Old examples frequently fail to compile against modern `winit`.
- **Web target caveats.** In 2021 browser WebGPU was behind flags; most web deployments actually ran on the WebGL fallback with its reduced feature set (no compute shaders, storage-buffer limits). Assumptions baked into old web builds may not hold on today's WebGPU-enabled browsers.
- **MPL-2.0 licensing.** The wrapper is file-level copyleft (Mozilla Public License 2.0), not MIT/Apache like much of the Rust ecosystem. The successor `gfx-rs/wgpu` relicensed to MIT OR Apache-2.0, so licensing conclusions drawn from this repo do not carry forward.

## When to Use / When Not

**Use when:**
- You are reproducing or bisecting an old build that pinned `wgpu-rs` specifically.
- You are reading historical material and need to understand the pre-`wgpu-hal` architecture.

**Avoid when:**
- You are writing anything new — use the `wgpu` crate from `gfx-rs/wgpu`.
- You need current backend support, security fixes, WGSL spec conformance, or compatibility with a modern `winit`/`naga`.
- You want long-term-stable APIs — this snapshot predates most of WebGPU's stabilization.

## Alternatives

- gfx-rs/wgpu — the direct successor and correct default; the `wgpu` crate here is what `wgpu-rs` became.
- bevyengine/bevy — use instead when you want an engine, not a raw graphics API; Bevy renders on top of `wgpu`.
- ash — use instead when you want raw, unopinionated Vulkan bindings with no portability layer.
- vulkano — use instead when you want a safe Rust Vulkan wrapper (Vulkan-only, not portable to Metal/DX/Web).
- grovesNL/glow — use instead when you only target OpenGL/WebGL and want a thin GL abstraction.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-05-10 | Rust wrapper split out under gfx-rs, tracking the draft WebGPU API[^1]. |
| 0.x series | 2019–2021 | Frequent breaking releases as the WebGPU spec evolved; API tracked spec drafts, not native APIs. |
| archived | 2021-06-03 | Last commit; repo folded into `gfx-rs/wgpu` monorepo as the top-level `wgpu` crate[^2]. |

## References

[^1]: gfx-rs organization, `wgpu-rs` repository (archived). https://github.com/gfx-rs/wgpu-rs
[^2]: `gfx-rs/wgpu` — consolidated repository that absorbed `wgpu-rs`; the current `wgpu` crate lives here. https://github.com/gfx-rs/wgpu
[^3]: naga — shader translation library (WGSL/SPIR-V/GLSL) used by wgpu. https://github.com/gfx-rs/naga

## Tags

rust, webgpu, graphics, gpu, wgpu, wrapper, archived, vulkan, metal, cross-platform, wasm
