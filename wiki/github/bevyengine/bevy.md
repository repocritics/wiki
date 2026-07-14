# bevyengine/bevy

> A data-driven game engine in Rust built around an archetypal ECS — pre-1.0, refactored aggressively, and moving fast.

[GitHub repo](https://github.com/bevyengine/bevy) ·
[Official website](https://bevy.org) ·
[License: MIT OR Apache-2.0](https://github.com/bevyengine/bevy#license)

## Overview

Bevy is a general-purpose 2D/3D game engine written in Rust, created by Carter Anderson ("Cart") and first released in August 2020[^1]. Its organizing idea is the Entity Component System (ECS): game state lives in plain data components attached to entities, and behavior lives in free functions ("systems") that query for the components they care about. There is no scene-graph object hierarchy and no GameObject base class — the engine itself is assembled from the same plugin and ECS primitives that application code uses.

The engine is modular to an unusual degree. `DefaultPlugins` is a bundle you can decompose; rendering, input, audio, UI, and windowing are each independent crates you can omit or replace. This composability is Bevy's defining strength and the source of its defining tension: the engine is still pre-1.0, ships breaking API changes roughly every three months on a "train" schedule[^2], and does not yet have a stable shipped editor[^3]. You get a clean, testable data-oriented architecture at the cost of API churn, sparse documentation in newer subsystems, and a lot of "build it yourself" for tooling that mature engines provide out of the box.

Bevy suits developers who want Rust's type system and performance for game or simulation work, are comfortable tracking a moving target, and value architecture over a turnkey content pipeline. It is a poor fit for teams that need to ship on a fixed deadline with a visual editor and asset store today.

## Getting Started

```sh
cargo add bevy
```

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position(Vec2);

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_systems(Startup, spawn)
        .add_systems(Update, move_things)
        .run();
}

fn spawn(mut commands: Commands) {
    commands.spawn(Camera2d);
    commands.spawn(Position(Vec2::ZERO));
}

fn move_things(time: Res<Time>, mut q: Query<&mut Position>) {
    for mut p in &mut q {
        p.0.x += 100.0 * time.delta_secs();
    }
}
```

For fast iterative builds, enable dynamic linking during development: `cargo run --features bevy/dynamic_linking`. Do not ship release builds with it.

## Architecture / How It Works

**ECS storage.** Bevy uses an archetype-based ECS: entities with the same set of component types are grouped into an archetype where each component is stored in a contiguous column. Iterating a query walks matching archetypes, which is cache-friendly. Components that are added/removed frequently can opt into sparse-set storage instead to avoid archetype moves. This layout is fast for iteration but makes structural changes (adding/removing components) relatively expensive, which is why mutations are usually deferred through `Commands` and applied at sync points.

**Scheduling.** Systems are scheduled into a directed graph and run in parallel across threads based on their data access — two systems that borrow the same component mutably cannot run concurrently, and the scheduler infers this from function signatures. Ordering between systems is explicit (`.before()`, `.after()`, system sets); unordered systems with conflicting access are an "ambiguity" the engine can report. The scheduler was reworked into its current "stageless" form in the 0.10 cycle[^4].

**Rendering.** Bevy renders through `wgpu`, so the same code targets Vulkan, Metal, DX12, and WebGPU/WebGL2. The render layer runs as a separate ECS world with its own extract/prepare/queue/render phases each frame, pipelined against the main-world simulation. Shaders are authored in WGSL.

**Reflection.** `bevy_reflect` provides runtime type information — the basis for scene serialization, the (in-progress) editor, and inspector tooling. **Assets** load asynchronously with hot-reloading support. Two ergonomic pillars landed recently: *required components* (0.15) let a component declare others it pulls in on spawn, replacing the older bundle pattern; *entity relationships* (0.16) add first-class linked entities (e.g. parent/child) maintained by the ECS.

## Production Notes

**Compile times are the dominant complaint.** A clean build of Bevy plus dependencies is large, and Rust's codegen is slow. Standard mitigations: `bevy/dynamic_linking` for dev iteration, a faster linker (`lld` or `mold`), the cranelift codegen backend on nightly for debug builds, and `cargo check` in the edit loop. Cold CI builds of a non-trivial Bevy project routinely run in the multi-minute range.

**Breaking changes every release.** Each ~3-month release ships a migration guide, but upgrades across several versions are real work: renamed APIs, reorganized crates, and reworked subsystems (the renderer, the asset system, and the scheduler have each been rewritten during the 0.x line). Pin a version and budget time to migrate rather than tracking `main`.

**No official editor yet.** Scene editing, inspectors, and asset management are community crates (`bevy-inspector-egui`, `bevy_editor_pls`, and others) rather than a first-party tool. An official editor is a stated goal and under active development, but you should not plan around it shipping on any particular date. The Bevy Remote Protocol (BRP) exposes the ECS world over JSON-RPC for external tooling.

**Ecosystem maturity varies.** Third-party plugins are abundant (physics via `avian`/`bevy_rapier`, tilemaps, UI kits, networking) but many track Bevy versions with a lag, so immediately after a release the plugin ecosystem is briefly behind. Audio, UI, and animation are usable but less complete than in Godot or Unity.

**Web builds work** via WASM + WebGL2/WebGPU, but binary size and load time need attention (`wasm-opt`, asset streaming). WebGPU support in browsers is still uneven across platforms.

## When to Use / When Not

**Use when:**
- You want Rust end-to-end and value a data-oriented, testable architecture.
- Your project is simulation-heavy or benefits from parallel ECS iteration.
- You are comfortable upgrading across breaking releases and assembling tooling.
- You want a modular engine you can strip down (e.g. headless server, custom render pipeline).

**Avoid when:**
- You need a mature visual editor and asset store to ship this quarter.
- You have a hard deadline and cannot absorb ~quarterly breaking migrations.
- Your team is new to Rust and the borrow checker plus API churn would compound.
- You need battle-tested console export pipelines today.

## Alternatives

- godotengine/godot — full-featured engine with a mature editor and GDScript; use it when you need to ship now with visual tooling rather than assemble it.
- FyroxEngine/Fyrox — Rust engine that ships a working scene editor; use it when you want Rust plus first-party editing over Bevy's modularity.
- not-fl3/macroquad — minimal immediate-mode Rust game library; use it for small 2D games where an ECS and plugin system are overkill.
- amethyst — the earlier Rust ECS engine, now discontinued with its maintainers pointing users to Bevy; do not start new projects on it.
- Unity / Unreal — proprietary engines with complete content pipelines; use them when tooling, asset ecosystem, and console support outweigh wanting an open-source Rust stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2020-08 | First public release; ECS, renderer, initial plugin system[^1]. |
| 0.6 | 2022-01 | New renderer, WGSL shaders; the "train" release schedule announced[^2]. |
| 0.10 | 2023-03 | Stageless scheduling rework[^4]. |
| 0.12 | 2023-11 | Asset system v2, deferred rendering. |
| 0.14 | 2024-07 | Continued rendering and ECS refinements. |
| 0.15 | 2024-11 | Required components replace the bundle pattern. |
| 0.16 | 2025-04 | First-class entity relationships; GPU-driven rendering work. |

Bevy has not reached 1.0; every 0.x release may include breaking changes.

## References

[^1]: Carter Anderson, "Introducing Bevy" — 2020-08-10. https://bevy.org/news/introducing-bevy/
[^2]: Bevy blog, "Bevy 0.6" (the ~3-month "train" release schedule). https://bevy.org/news/bevy-0-6/#the-train-release-schedule
[^3]: Bevy README "Warning": early stages of development, breaking changes ~every 3 months, sparse docs. https://github.com/bevyengine/bevy#warning
[^4]: Bevy blog, "Bevy 0.10" (stageless scheduling). https://bevy.org/news/bevy-0-10/

## Tags

rust, game-engine, ecs, gamedev, 2d, 3d, data-oriented, wgpu, cross-platform, open-source
