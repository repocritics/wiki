# godotengine/godot

A free, open-source 2D and 3D game engine — the leading OSS alternative to Unity and Unreal, especially after Unity's 2023 pricing controversy drove developer migration.

## What it is

A multi-platform game engine that ships its own editor (Godot Editor), a node-based scene system, two scripting languages (GDScript and C#, plus C++ for native modules), and exporters for Windows / macOS / Linux / Android / iOS / Web / consoles (via third-party deployers). Distinguishes itself from Unity / Unreal by being fully open-source under MIT — no per-developer licensing, no royalties, no telemetry. Maintained by the Godot Foundation with significant corporate sponsorship.

## Key features

- Built-in 2D and 3D engines (the 2D pipeline is genuinely 2D, not a 3D engine pretending).
- Node + Scene architecture — compose game objects as trees of nodes that can be instanced as reusable scenes.
- GDScript (Python-like, native to Godot), C# (.NET), and C++ (GDExtension) scripting.
- Vulkan (Forward+ / Mobile) and OpenGL renderers; web export via WebGL/WebGPU.
- Visual shader editor + native shader language.
- Editor is itself a Godot game — fully self-hostable on any platform that runs the engine.
- MIT-licensed.

## Tech stack

- C++ primary for the engine core.
- GDScript (Godot's first-class scripting language) and C# (Mono / .NET).
- Vulkan + OpenGL for rendering.
- Distributed as a single ~50MB editor binary plus exporter binaries.

## When to reach for it

- You want a free, OSS game engine without per-developer fees or revenue royalties.
- You're building a 2D game — Godot's 2D pipeline is genuinely first-class.
- You want an editor that runs on Linux as a first-class platform.
- You left Unity for licensing or business reasons and want the most production-mature OSS alternative.

## When *not* to reach for it

- You need AAA-tier 3D graphics — Unreal's renderer is significantly ahead for high-end 3D.
- You depend on a specific Unity asset / plugin ecosystem — Godot's marketplace is smaller and not always interchangeable.
- You're targeting consoles directly — Godot supports them via third-party publishers, not first-party SDKs.

## Maturity signal

112k stars, 25k forks, MIT, last push 2026-06-01. 12-year-old project under the Godot Foundation. Open-issues count of 18,336 is very high in absolute terms but proportional to the engine's surface area — feature requests and per-platform reports across 6 export targets accumulate. Major sponsor backing (Godot Foundation, W4 Games, Ramatak, AppLovin, plus community donations) signals sustained investment. The 4.x release line is the modern path; 3.x remains supported for projects pinned there.

## Alternatives

- Unity — use when you want the largest ecosystem and don't object to the licensing terms.
- Unreal Engine — use when you want AAA-tier 3D graphics and you can navigate UE's licensing.
- `oven-sh/bun` + custom engine — out of scope; sometimes asked.
- Bevy (Rust) — use when you want code-first ECS with no editor (yet).
- LÖVE (Lua) — use when you want a small 2D framework rather than a full engine.

## Notes

The 2023 Unity pricing controversy + Godot's MIT license drove a significant migration wave; Godot's adoption surged proportionally. The Foundation governance model isolates the engine from vendor-control risk — that's the project's most-strategic property for long-term game projects. Console export goes through paid third-party publishers (W4 Games' W4 Console, Ramatak) since console SDKs themselves require NDA-gated licensing.

## Tags

godot, game-engine, c-plus-plus, gamedev, 2d, 3d, cross-platform, vulkan, opengl, mit-license, open-source
