# gfx-rs/wgpu

> A cross-platform, safe, pure-Rust graphics API — the WebGPU standard implemented as a native library, and the GPU backend inside Firefox, Servo, and Deno.

[GitHub repo](https://github.com/gfx-rs/wgpu) ·
[Official website](https://wgpu.rs) ·
[License: Apache-2.0 OR MIT](https://github.com/gfx-rs/wgpu/blob/trunk/LICENSE.APACHE)

## Overview

wgpu is an implementation of the WebGPU API written in Rust[^1]. Despite the "web" name, it is a fully native library: it runs on Vulkan, Metal, and D3D12 as first-class backends, on OpenGL/GLES as best-effort, and — when compiled to WebAssembly — on top of a browser's WebGPU implementation or WebGL2. The same Rust code targeting the WebGPU API therefore runs on desktop, mobile, and the browser without a rewrite. This portability is the project's reason to exist and its central design constraint: every feature must map onto the lowest-common-denominator of five very different graphics APIs.

wgpu is not a hobby project. It is the engine behind the WebGPU integration in Firefox, Servo, and Deno[^2], which means its API surface and conformance are driven by the actual WebGPU specification and the WebGPU Conformance Test Suite (CTS), not by convenience. It is also the most-used GPU abstraction in the Rust game and graphics ecosystem — Bevy, egui/eframe, and many renderers build on it.

The defining tension is **spec-tracking versus stability**. The WebGPU and WGSL specifications are still formally in "Working Draft" status[^3], so wgpu changes its public API on a fixed quarterly cadence, and every release is a breaking release. Teams get a modern, safe, portable GPU API; they pay for it with a steady stream of migration work.

## Getting Started

```bash
cargo add wgpu
cargo add pollster    # a minimal async executor for the request_* calls
```

```rust
// Acquire an adapter (a physical GPU) and a logical device + queue.
// Exact signatures shift between quarterly releases; check docs.rs for your version.
fn main() {
    pollster::block_on(run());
}

async fn run() {
    let instance = wgpu::Instance::default();

    let adapter = instance
        .request_adapter(&wgpu::RequestAdapterOptions::default())
        .await
        .expect("no compatible GPU adapter found");

    let (device, queue) = adapter
        .request_device(&wgpu::DeviceDescriptor::default())
        .await
        .expect("failed to create device");

    println!("using {:?}", adapter.get_info());
    let _ = (device, queue); // build pipelines, buffers, command encoders from here
}
```

Shaders are written in WGSL, the WebGPU Shading Language. For a full triangle-drawing walkthrough, the community tutorial *Learn Wgpu*[^4] is the standard starting point.

## Architecture / How It Works

wgpu is a workspace of layered crates, not a monolith:

- **`wgpu`** — the safe, idiomatic public API. This is what applications depend on. It mirrors the WebGPU spec's object model: `Instance` → `Adapter` → `Device`/`Queue` → pipelines, buffers, textures, bind groups, command encoders.
- **`wgpu-core`** — the validation and state-tracking layer. It enforces the WebGPU safety rules (resource lifetimes, usage flags, bind-group compatibility) so that invalid GPU work is rejected in Rust rather than causing undefined behavior on the device. This is the layer Firefox embeds.
- **`wgpu-hal`** — the unsafe, thin hardware abstraction layer. One backend module per native API (Vulkan, Metal, D3D12, GLES). It is deliberately close to the metal and is *not* meant to be used directly by applications, though it is public for advanced/embedding use.
- **`wgpu-types`** — plain data types (enums, flags, descriptors) shared across the stack with no logic.

Shader translation is handled by **Naga**[^5], a sibling crate in the same repo. Naga parses WGSL (and GLSL/SPIR-V) into an intermediate representation and emits the shading language each backend needs: SPIR-V for Vulkan, MSL for Metal, HLSL for D3D12, GLSL for OpenGL. Naga replaced the earlier `spirv-cross` C++ dependency, which is why wgpu can honestly call itself "pure-Rust." When you run in a browser without the `webgl` feature, WGSL is instead passed straight through to the browser's own implementation, so which WGSL features work there depends on the browser, not on Naga.

The safety story is layered: `wgpu-hal` contains the `unsafe` code and per-driver workarounds; `wgpu-core` turns that into a validated, panic-or-error API; `wgpu` exposes it as safe Rust. Bugs and driver quirks are concentrated in `wgpu-hal`, which is where most platform-specific pain lives.

## Production Notes

**Every release is a breaking release.** wgpu ships roughly every three months and treats the API as unstable until WebGPU itself stabilizes. Upgrading across two or three versions typically means non-trivial code changes — descriptor fields get added, `request_adapter`/`request_device` signatures change (Option vs Result, added parameters), and enum variants come and go. Pin your version and budget for periodic migration rather than tracking `trunk`.

**MSRV.** As of current releases, building `wgpu` requires Rust **1.87**; running the repo's own tests and examples requires **1.93**[^6]. The project ties `wgpu-core`'s MSRV to Firefox's and `wgpu`'s to Servo's, so it will not casually outrun the browsers that embed it. An MSRV bump is itself considered a breaking change.

**Backend selection and driver reality.** "Cross-platform" does not mean "identical everywhere." The `WGPU_BACKEND` and `WGPU_ADAPTER_NAME` environment variables let you force a backend/adapter, which you will need when a specific driver misbehaves. Vulkan on macOS/iOS runs through MoltenVK; OpenGL on Apple platforms needs ANGLE. The D3D12 path can use `dxc`, `static-dxc`, or the legacy `fxc` shader compiler (via `WGPU_DX12_COMPILER`) — `dxc` needs `dxcompiler.dll` present or it silently falls back to `fxc`.

**Downlevel limits.** The GLES/WebGL2 backends do not support the full feature set (no compute shaders on WebGL2, limited storage buffers, texture-format gaps). Code that runs on Vulkan/Metal/D3D12 can fail at device-creation time on GL. Query `Features` and `Limits` and degrade gracefully rather than assuming a capability exists.

**Validation cost and error surface.** `wgpu-core` validation catches misuse early but is not free; the WebGPU model also uses an asynchronous, callback-based error scope, so a bad command may surface later than the call that caused it. Enable validation layers during development and expect some errors to be reported out of band.

**Async that isn't really async.** `request_adapter`, `request_device`, and buffer mapping are `async` to match the web model, but on native they usually resolve immediately. Pulling in a full async runtime is unnecessary — `pollster::block_on` is the common lightweight choice.

## When to Use / When Not

**Use when:**
- You want one GPU codebase that runs on Windows, Linux, macOS, iOS, Android, and the web.
- You need a memory-safe GPU API — validation and Rust's type system eliminate whole classes of graphics UB.
- You are building a renderer, game, or visualization tool in the Rust ecosystem (Bevy, egui, and friends already assume wgpu).
- You want to target WebGPU in the browser today while keeping a native fallback.

**Avoid when:**
- You need bleeding-edge, vendor-specific GPU features (ray-tracing extensions, mesh shaders) with no portability compromise — a direct Vulkan/D3D12/Metal binding gives you everything the driver exposes. wgpu is gaining some of these behind feature flags but lags the native APIs.
- You cannot absorb quarterly breaking upgrades or you need a frozen, decade-stable API.
- You are shipping only to a single platform and want the absolute thinnest possible abstraction and maximum control.
- Your workload is pure GPU compute at scale where a CUDA/ROCm or raw-Vulkan-compute path is a better fit.

## Alternatives

- gfx-rs/wgpu-native — C bindings to this same library; use it when you want WebGPU from C/C++ or another language rather than Rust.
- Vulkano (vulkano-rs/vulkano) — safe Rust bindings to Vulkan specifically; use it when you want full Vulkan power and safety and don't need portability to Metal/D3D12/web.
- ash (ash-rs/ash) — thin unsafe Rust Vulkan bindings; use it when you want zero abstraction and every Vulkan feature and will manage safety yourself.
- SDL / bgfx (bkaradzic/bgfx) — mature C/C++ cross-platform graphics abstractions; use them when you're outside the Rust ecosystem or need a proven C++ renderer.
- glium / miniquad — lighter OpenGL-era Rust libraries; use them when WebGL/OpenGL is enough and you want a smaller dependency than the full WebGPU stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019 | `wgpu-rs` begins as a Rust wrapper over the `gfx-hal` abstraction layer. |
| 0.7 | 2021 | Naga replaces `spirv-cross`, moving shader translation to pure Rust[^5]. |
| 0.10–0.11 | 2021 | Migration from external `gfx-hal` to the in-house `wgpu-hal` backend layer. |
| 0.19 | 2024-01 | Late 0.x line; quarterly release rhythm established. |
| 22.0 | 2024-07 | Version scheme drops the leading `0.`; major version now tracks the real release count. |
| 30 | 2026 | Current release line; three-month breaking-release cadence continues[^1]. |

## References

[^1]: wgpu README and repository, gfx-rs/wgpu. https://github.com/gfx-rs/wgpu
[^2]: Firefox, Servo, and Deno use wgpu as the core of their WebGPU implementations, per the project README. https://github.com/gfx-rs/wgpu#readme
[^3]: WebGPU specification (W3C Working Draft) and WGSL specification. https://www.w3.org/TR/webgpu/ · https://gpuweb.github.io/gpuweb/wgsl/
[^4]: "Learn Wgpu" community tutorial, sotrh. https://sotrh.github.io/learn-wgpu/
[^5]: Naga shader translation crate, in-tree at gfx-rs/wgpu. https://github.com/gfx-rs/wgpu/tree/trunk/naga
[^6]: wgpu MSRV policy (wgpu 1.87; repo tests/examples 1.93), README. https://github.com/gfx-rs/wgpu#msrv-policy

## Tags

rust, graphics, gpu, webgpu, vulkan, metal, d3d12, opengl, cross-platform, wgsl, rendering, wasm
