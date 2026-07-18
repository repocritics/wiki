# bkaradzic/bgfx

> Cross-platform, graphics-API-agnostic rendering library — "bring your own engine/framework," it brings the GPU abstraction and nothing else.

[GitHub repo](https://github.com/bkaradzic/bgfx) ·
[Official docs](https://bkaradzic.github.io/bgfx/overview.html) ·
[License: BSD-2-Clause](https://github.com/bkaradzic/bgfx/blob/master/LICENSE)

## Overview

bgfx is a rendering library started by Branimir Karadžić in 2012[^1] that abstracts a dozen graphics backends — Direct3D 11/12, Metal, Vulkan, OpenGL 2.1/3.1+, OpenGL ES 2/3.1, WebGL 1/2, WebGPU (Dawn native only), and GNM for licensed PS4 developers — behind one submission-based API. It is deliberately *not* an engine: no scene graph, no material system, no asset loading, no windowing. You hand it a native window handle and draw calls; it handles backend selection, state management, and draw sorting. The C++ API is mirrored by an auto-generated C99 API, which is why bindings exist for C#, Rust, Zig, D, Lua, Java (LWJGL3), Python and more[^2].

The defining tradeoff is scope discipline versus convenience. Compared to Filament or a full engine, you write far more code before pixels appear (shaders must be compiled offline with bgfx's own toolchain, meshes loaded yourself, windows created yourself). In exchange you get a small, stable-ish surface that has quietly outlived several "modern renderer" projects and ships in serious production software: Minecraft Bedrock's RenderDragon renderer[^3], MAME, Babylon Native, Football Manager's match engine, and dozens of mobile titles.

It is a single-BDFL project — Karadžić reviews essentially everything — which cuts both ways: coherent API design over 14 years, but a real bus factor and a "read the source, then ask on Discord" support model. Activity is healthy as of mid-2026 (last push July 2026; ~17.3k stars, ~2.1k forks, 288 open issues).

## Getting Started

bgfx requires its two sibling repos, `bx` (base library) and `bimg` (image library), cloned side by side. The native build system is GENie (a premake fork, bundled in bx); a community-maintained CMake wrapper exists as `bkaradzic/bgfx.cmake`[^4].

```bash
git clone https://github.com/bkaradzic/bx.git
git clone https://github.com/bkaradzic/bimg.git
git clone https://github.com/bkaradzic/bgfx.git
cd bgfx
make linux-release64        # or: vs2022 project gen on Windows, osx-* on macOS
```

Minimal frame loop (windowing via GLFW/SDL/native — not shown, bgfx only wants the handle):

```cpp
bgfx::Init init;
init.platformData.nwh = nativeWindowHandle;   // HWND / NSWindow* / X11 window ...
init.resolution.width  = 1280;
init.resolution.height = 720;
init.resolution.reset  = BGFX_RESET_VSYNC;
bgfx::init(init);                             // backend auto-selected per platform

bgfx::setViewClear(0, BGFX_CLEAR_COLOR | BGFX_CLEAR_DEPTH, 0x303030ff);
bgfx::setViewRect(0, 0, 0, 1280, 720);

while (running) {
    bgfx::touch(0);       // keep view 0 alive even with no draws
    bgfx::frame();        // submit frame, kick render thread
}
bgfx::shutdown();
```

Shaders are written in bgfx's GLSL-flavored shader language and compiled offline with `shaderc` into per-backend binaries; there is no runtime shader compilation from source[^5].

## Architecture / How It Works

- **Views and sort keys.** Rendering is organized into up to 256 "views" (roughly: render passes with their own framebuffer, viewport, and clear state). Draw calls submitted to a view are encoded into 64-bit sort keys and reordered — by program, depth, or strict sequence per view — to minimize state changes. You do not control raw command order unless you opt into sequential mode[^6].
- **Two-thread model.** By default bgfx runs API calls on your thread and backend execution on an internal render thread; `bgfx::frame()` is the synchronization point. This buys parallelism at the cost of one frame of latency between submit and render. Multiple submission threads are supported via explicit `Encoder` objects. Single-threaded operation is a compile-time/init choice.
- **Opaque handles, compile-time limits.** Resources (vertex/index buffers, textures, shaders, framebuffers) are 16-bit handles. Maximum counts for everything — draw calls per frame, views, texture samplers — are `BGFX_CONFIG_*` compile-time constants; exceeding them is a rebuild, not a config change.
- **Shader toolchain.** `shaderc` cross-compiles the bgfx shader dialect (annotated with `$input`/`$output` and a `varying.def.sc` file) to DXBC/DXIL, SPIR-V, MSL, or GLSL variants. `texturec`, `geometryc`, and `texturev` round out the offline pipeline.
- **Coupling.** bgfx is inseparable from `bx` (allocators, containers, platform layer — it deliberately avoids the C++ STL) and `bimg`. The examples ship with an `entry` framework (windowing, input, imgui integration) that is example scaffolding, not part of the library; new users routinely mistake it for a supported app framework.
- **Capability querying.** Backend feature disparity (compute, instancing, texture formats, MRT counts) is surfaced through `bgfx::getCaps()` at runtime — portable code must branch on caps rather than assume the union of features.

## Production Notes

- **No semver, no stable releases.** bgfx uses rolling `v1.<API-version>.<commit>` tags off `master`; breaking API changes land as API-version bumps with terse commit messages. The standard practice is pinning a commit and reading the diff of `bgfx.h` + `examples` when upgrading. Budget real time for upgrades after long gaps.
- **Backend maturity is uneven.** D3D11 and OpenGL are the longest-baked paths; Metal is solid; Vulkan reached parity later and historically trailed on edge cases; the WebGPU backend is explicitly Dawn-native-only and experimental. Test on every backend you ship — identical API calls can have different performance cliffs (uniform updates and dynamic buffers especially).
- **Shader pipeline friction is the #1 onboarding complaint.** Offline-only compilation means shaderc must be wired into your asset build for every backend/profile matrix entry, and errors are frequently reported by the downstream translator (glslang/SPIRV-Cross/fxc) in translated code, not your source.
- **One frame of latency** from the multithreaded submit model. Fine for games; relevant for latency-sensitive tools. `bgfx::frame()` stalls if the render thread falls behind — profile with the built-in stats overlay (`BGFX_DEBUG_STATS`) before blaming your code.
- **Compile-time limits bite late.** Hitting the default draw-call or transient-buffer ceilings mid-project means recompiling the library with adjusted `BGFX_CONFIG_*` defines — awkward if you consume bgfx as a prebuilt binary.
- **Debugging support is decent:** RenderDoc integration works well on D3D/Vulkan, debug names propagate to native API objects, and the debug-text overlay is genuinely useful. Metal/Xcode capture works but with less annotation.
- **Support channel is Discord + GitHub Discussions**, and the documentation covers the API surface but thins out on architecture recipes; expect to read example source (there are 40+ examples, which are effectively the real documentation).

## When to Use / When Not

**Use when:**
- You are building your own engine or framework and want the GPU portability layer solved without adopting someone's scene model.
- You need one codebase spanning desktop, mobile, web (WASM/WebGL), and consoles with a proven track record.
- You value a small C-compatible API you can bind from another language (the C99 API is first-class, not an afterthought).
- Long-term maintenance matters more than novelty — 14 years of continuous development is rare in this category.

**Avoid when:**
- You want materials, lighting, or a render graph out of the box — bgfx gives you draw submission, not a renderer. Filament or an engine is the right layer.
- You need bleeding-edge explicit-API features (mesh shaders, ray tracing, bindless-everything) — the lowest-common-denominator abstraction lags the newest hardware features by design.
- Your team expects semver releases, LTS branches, or vendor support contracts.
- You are latency-critical and cannot tolerate the extra frame from the threaded submit model without careful configuration.

## Alternatives

- google/filament — use instead when you want a full PBR renderer (materials, lights, IBL) rather than a raw submission API.
- floooh/sokol — use instead when you want a much smaller header-only C abstraction and can accept fewer backends and features.
- gfx-rs/wgpu — use instead when you are in Rust or want the WebGPU API shape as your portable abstraction.
- DiligentGraphics/DiligentEngine — use instead when you want a modern explicit-API-style abstraction (PSOs, command lists) closer to D3D12/Vulkan semantics.
- ConfettiFX/The-Forge — use instead when you need console-grade explicit control and are comfortable with a framework-heavier codebase.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2012-04 | Initial development; original backends included D3D9 and OpenGL[^1]. |
| Metal backend | ~2015 | Contributed backend; iOS/macOS support without OpenGL deprecation risk. |
| Vulkan backend | 2016→ | Landed early, reached practical parity with D3D11 over subsequent years. |
| WebGPU backend | ~2020 | Experimental, Dawn native only. |
| D3D9 backend removed | ~2021 | Minimum Windows path became D3D11. |
| Rolling releases | ongoing | `v1.<API>.<commit>` tags from `master`; no semver, no LTS. |

## References

[^1]: bgfx documentation — Overview. https://bkaradzic.github.io/bgfx/overview.html
[^2]: bgfx README — language bindings list. https://github.com/bkaradzic/bgfx#languages
[^3]: Minecraft attribution page (lists bgfx). https://www.minecraft.net/en-us/attribution
[^4]: bgfx.cmake — community CMake build. https://github.com/bkaradzic/bgfx.cmake
[^5]: bgfx documentation — Tools (shaderc, texturec, geometryc). https://bkaradzic.github.io/bgfx/tools.html
[^6]: bgfx documentation — Internals. https://bkaradzic.github.io/bgfx/internals.html

## Tags

c-plus-plus, graphics, rendering, gamedev, cross-platform, vulkan, metal, direct3d, opengl, webgl, middleware, low-level
