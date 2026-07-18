# hexops/mach

> A Zig game engine and graphics toolkit that builds and cross-compiles from
> any OS to any OS with `zig build` alone — no SDKs, no system dependencies.

[GitHub repo](https://github.com/hexops/mach) ·
[Official website](https://machengine.org) ·
License: MIT OR Apache-2.0 (code), CC-BY-4.0 (docs/assets)[^1]

## Overview

Mach is a game engine and graphics toolkit written in Zig, started in 2021 by
Stephen Gutekanst (Hexops)[^2]. Its core promise is unusual among engines:
`zig build` on any host produces native binaries for Windows, Linux, and macOS
without Visual Studio, Xcode, or platform SDKs — Mach vendors the system
headers and libraries Zig's cross-compilation needs. It targets games,
visualizations, and desktop GUI apps, and is modular: you can consume just
the windowing/input/GPU core and ignore the engine layer entirely.

The defining tension is ambition versus surface area. Mach builds its own GPU
abstraction (sysgpu, replacing an earlier dependency on Google's Dawn WebGPU
implementation)[^3], its own audio layer (sysaudio), its own object/entity
system, and vendors its own toolchain artifacts — all sustained by a small
contributor core led by one primary developer, funded through sponsorship.
Combined with Zig itself being pre-1.0, Mach is explicitly pre-stability:
APIs are rewritten between minor versions, and the project frames itself as
building toward a "Mach 1.0" rather than being production-ready today.

At 4,800 stars and 210 forks it is the most visible Zig-native game engine.
Tag v0.5.0 landed in April 2026 and the mirror was pushed May 2026, so it is
actively maintained — but primary development moved to the self-hosted forge
at code.hexops.org; the GitHub repo is explicitly a mirror "purely for
visibility"[^4], so GitHub activity counts understate real project state.

## Getting Started

Mach does not track stable Zig — it pins a specific "nominated" Zig nightly,
published on machengine.org; install exactly that version or the build will
fail[^5].

```bash
git clone https://github.com/hexops/mach
cd mach
zig build -l              # list build steps, including example run targets
zig build run-<example>   # run one of the bundled examples
```

To depend on Mach from your own project, add it to `build.zig.zon` with the
exact package URL/hash the docs pin for your nominated Zig version:

```zig
.dependencies = .{
    .mach = .{
        .url = "https://pkg.machengine.org/mach/<version>.tar.gz",
        .hash = "...", // from `zig fetch --save <url>`
    },
},
```

## Architecture / How It Works

Mach is layered, and the layers are consumable independently:

- **Core** — window creation, input, and a GPU device, in a single package.
  This is the "bring your own engine" entry point: many users take only this
  and write their own renderer on top.
- **sysgpu** — Mach's cross-platform graphics abstraction, written in Zig,
  with a WebGPU-shaped API targeting Vulkan, Metal, D3D12, and OpenGL
  backends[^3]. Earlier versions bound Google's Dawn (C++ WebGPU) via
  prebuilt binaries; sysgpu replaced it to remove the C++ toolchain from the
  dependency graph, trading Dawn's maturity and conformance testing for a
  fully Zig-native stack the project controls.
- **sysaudio** — low-level cross-platform audio I/O.
- **Object system** — Mach's entity/component layer. The original ECS design
  was replaced with a simpler "objects" model during the 0.4 era; apps are
  composed of modules whose systems are wired together at comptime, so module
  composition is checked by the compiler rather than a runtime registry.
- **Vendored dependencies** — Mach maintains forks/binaries of system
  libraries (X11/Wayland headers, font libraries, etc.) so that
  cross-compilation needs nothing from the host OS.

Historically the project was ~dozens of `mach-*` repositories (mach-core,
mach-gpu, mach-glfw, bindings); these were consolidated into the single
`hexops/mach` monorepo around v0.3 (early 2024), simplifying versioning but
breaking consumers of the standalone packages. There is no editor: Mach is
code-first, and scenes, assets, and tooling are your problem, with in-repo
examples serving as the primary documentation alongside machengine.org.

## Production Notes

- **You cannot use stable Zig.** The nominated-Zig scheme means CI, team,
  and every dependency must agree on one specific Zig nightly; mixing a
  system Zig with Mach is the most common first-hour failure[^5].
- **API churn is real, not theoretical.** ECS → object system, Dawn →
  sysgpu, multi-repo → monorepo: each transition invalidated third-party
  tutorials. Expect most non-official material older than a year to be
  wrong, and budget migration work for every version bump.
- **sysgpu is young.** Dawn and wgpu have years of conformance testing;
  sysgpu does not. Graphics bugs you hit may be engine bugs, and validation/
  debug tooling is thinner than in established WebGPU implementations.
- **Bus factor.** One lead developer and a small contributor base carry an
  engine-sized scope, funded by sponsorship. The project is transparent
  about this, but it is a real risk for a multi-year game project.
- **GitHub is a mirror.** Issues, PRs, and current activity live on
  code.hexops.org[^4]; the 167 open issues visible on GitHub are a stale
  snapshot, and contributing means engaging with the self-hosted forge.
- **Platform reality check.** Desktop (Windows/Linux/macOS) is the supported
  path. Mobile and web have been stated goals for years but should be treated
  as experimental — verify current status before committing to those targets.
- **Where it genuinely delivers:** clean-machine builds. Clone + correct Zig
  + `zig build` producing a cross-compiled game binary with zero system setup
  is a workflow essentially no C++ engine matches.

## When to Use / When Not

**Use when:**
- You are committed to Zig and want windowing/input/GPU without writing
  platform backends yourself.
- Cross-compiled, zero-system-dependency builds matter (all desktop OSes
  from one CI runner).
- You want a modular toolkit — taking Core alone and building your own
  engine on top is an intended use.
- You are comfortable reading engine source as documentation.

**Avoid when:**
- You need to ship a commercial game on a deadline — pre-1.0 churn in both
  Mach and Zig will cost you.
- You want an editor, asset pipeline, or batteries-included workflow
  (Godot/Unity territory).
- You need mobile or console targets today, or your team cannot manage a
  pinned Zig nightly.

## Alternatives

- bevyengine/bevy — use instead when you want a comparable code-first,
  ECS-driven engine with a much larger ecosystem and accept Rust.
- godotengine/godot — use instead when you need a mature editor, asset
  pipeline, and shipping track record.
- raysan5/raylib — use instead when you want a simple, stable C API for
  learning or small games (usable from Zig via bindings).
- libsdl-org/SDL — use instead when you only need the platform layer
  (window/input/audio) with two decades of hardening.
- zig-gamedev/zig-gamedev — use instead when you prefer assembling individual
  Zig gamedev libraries over adopting a framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1 | 2022-03-27 | First tagged release; GLFW windowing + Google Dawn (WebGPU) graphics. |
| v0.2.0 | 2023-08-14 | Split-package era: mach-core, mach-gpu, sysaudio, standalone bindings. |
| v0.3.0 | 2024-02-02 | Monorepo consolidation; work begins on sysgpu, the Zig-native graphics API[^3]. |
| v0.4.0 | 2024-07-08 | Dawn dependency dropped in favor of sysgpu; simpler object system replaces the ECS. |
| v0.5.0 | 2026-04-10 | Current tagged release; primary development now on code.hexops.org[^4]. |

## References

[^1]: hexops/mach LICENSE — code dual-licensed MIT OR Apache-2.0; docs, images, sounds, fonts, and models CC-BY-4.0. https://github.com/hexops/mach/blob/main/LICENSE
[^2]: Machengine.org — official documentation and project site. https://machengine.org
[^3]: Hexops devlog — sysgpu and engine development posts. https://devlog.hexops.com/
[^4]: hexops/mach README — "Development occurs on code.hexops.org — our GitHub mirror is purely for visibility." https://code.hexops.org/hexops/mach
[^5]: Machengine.org — nominated Zig versions (Mach requires a specific pinned Zig nightly). https://machengine.org/about/nominated-zig/

## Tags

zig, game-engine, graphics, gamedev, webgpu, cross-platform, gui, modular, native, pre-1.0
