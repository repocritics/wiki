# love2d/love

> A code-only 2D game framework for Lua — you write callbacks, it draws frames. No editor, no scene tree, no runtime tax.

[GitHub repo](https://github.com/love2d/love) ·
[Official website](https://love2d.org) ·
[License: Zlib](https://github.com/love2d/love/blob/main/license.txt)

## Overview

LÖVE (styled "LÖVE", commonly "love2d") is a C++ framework that embeds Lua and exposes a set of 2D game modules — graphics, audio, physics, input, filesystem — as Lua tables[^1]. You do not open an IDE or build a scene graph. You write a `main.lua` with `love.load`, `love.update(dt)`, and `love.draw` callbacks, point the `love` binary at the folder, and it runs. That minimalism is the entire product thesis and the reason it has outlasted most of its contemporaries.

The project predates its GitHub mirror by roughly a decade: first public 0.x releases date to 2008, originally hosted on Mercurial/Bitbucket; the `love2d/love` GitHub repository was created in 2019[^2]. It is developed by a small volunteer core (notably Alexander "slime" Szpakowski on the graphics layer) rather than a company, which shows in its cadence — major versions ship years apart, and there is no commercial hosting or upsell attached.

The defining tradeoff: LÖVE gives you a clean, fast, well-documented API surface and then stops. There is no built-in entity system, level editor, animation timeline, or asset pipeline. For a programmer who wants direct control this is liberating; for a team expecting Unity- or Godot-style tooling it is a wall. Everything above the framework — scene management, ECS, Tiled map loading, tweening — comes from community Lua libraries or code you write yourself. Notably, the maintainers explicitly refuse pull requests and bug reports produced with generative-AI tooling[^3].

## Getting Started

Install the runtime (Homebrew shown; Windows/Linux/mobile builds are on the releases page):

```bash
brew install --cask love    # macOS
# or download from https://love2d.org
```

A complete game is one file, `main.lua`:

```lua
function love.load()
    x, y = 100, 100
end

function love.update(dt)
    if love.keyboard.isDown("right") then x = x + 200 * dt end
end

function love.draw()
    love.graphics.setColor(1, 0.4, 0.2)   -- 0..1 range since 11.0
    love.graphics.circle("fill", x, y, 40)
    love.graphics.print("Hello LÖVE", 10, 10)
end
```

Run it by pointing the binary at the directory:

```bash
love .          # run the current folder
love game.love  # or run a packaged .love archive
```

## Architecture / How It Works

At the core is a C++ host that initializes SDL, creates a window and graphics context, and boots a LuaJIT (or plain Lua) VM. Each subsystem is a C++ module registered into the Lua state as a global table under `love.*`[^1]:

- **love.graphics** — the largest module. Immediate-mode drawing over OpenGL 3.3+ / OpenGL ES 3.0+, with Vulkan and Metal backends landing in the 12.0 line. Supports shaders (GLSL-derived), canvases (render targets), sprite batches, and meshes.
- **love.physics** — a thin binding over Box2D. Bodies, fixtures, joints, world stepping.
- **love.filesystem** — backed by PhysFS, sandboxed to the game directory and a per-game save directory. You cannot freely touch the host filesystem, which is deliberate.
- **love.audio** — OpenAL-backed sources; streaming and static playback, positional audio.
- **love.thread** — real OS threads, but each thread gets its own Lua state. Threads share nothing; they communicate through `Channel` objects (message queues). There is no shared mutable Lua memory.

The `love.run` function is the actual main loop — event pump, `love.update(dt)`, `love.draw`, buffer swap — and can be overridden wholesale if you need custom timing or a fixed timestep.

Distribution is a defining design choice. A game is packaged as a `.love` file, which is just a ZIP of your Lua source and assets. To ship a standalone executable you concatenate the `.love` onto the `love` binary ("fusing"); the runtime detects the appended archive and runs it. This is elegant and cross-platform but means **your source is trivially extractable** — a `.love` or fused binary can be unzipped by anyone. There is no built-in code protection.

The main branch tracks the unreleased next major version and is explicitly documented as unstable[^3]; released majors get their own maintenance branches for patch releases.

## Production Notes

**LuaJIT is the performance story, and it has limits.** LÖVE ships LuaJIT on most platforms, which makes hot Lua loops fast — but LuaJIT is Lua 5.1 semantics. You do not get Lua 5.3 integer division or native bitwise operators; bit operations go through LuaJIT's `bit` library. Code written against mainline Lua 5.3/5.4 may not port cleanly.

**Garbage-collection pauses are yours to manage.** Frame-time spikes from Lua GC are the most common performance complaint. Teams shipping smooth games routinely tune `collectgarbage("step", ...)` per frame or set the GC to incremental mode rather than relying on defaults.

**The 11.0 color-range break is the canonical upgrade landmine.** Before 11.0, colors were 0–255 integers; 11.0 switched every color API to 0–1 floats[^4]. This silently broke essentially all pre-11.0 code and community tutorials — a `setColor(255, 0, 0)` now reads as clamped white. When copying old snippets, check the version.

**No scene system means you own architecture.** Common community dependencies fill the gap: `bump.lua` (AABB collision), `hump` (gamestate, timers, vectors), `anim8` (spritesheet animation), and `STI` (Simple Tiled Implementation) for Tiled maps. None ship with LÖVE; pin versions yourself.

**Mobile and web are second-class but real.** Android builds go through the separate `love-android` project; iOS requires macOS, Xcode, an Apple developer account, and the `love-apple-dependencies` bundle. A web target exists via `love.js` (Emscripten) but carries large download sizes and performance caveats. Desktop is the first-class path.

**12.0 is not released.** The SDL3 migration, Vulkan/Metal backends, and graphics API changes on `main` are attractive but unshipped as of mid-2026; production work should stay on the 11.x line (11.5 is the current stable) and expect a migration pass when 12.0 lands.

LÖVE's viability as a serious tool was underscored by Balatro (2024), a commercially successful poker roguelike built entirely on the framework[^5] — evidence that "code-only, no engine" scales to a finished, shipped, high-volume title.

## When to Use / When Not

**Use when:**
- You are a programmer who wants to write a 2D game in code with a clean API and no editor overhead.
- You want a tiny, fast, dependency-light runtime and full control of the game loop.
- You are prototyping game mechanics quickly, or teaching programming through games.
- Your target is primarily desktop (Windows/macOS/Linux).

**Avoid when:**
- Your team expects a visual editor, scene tree, or asset pipeline (use Godot or Unity).
- Mobile or web is your primary platform and you need a smooth packaging/store story.
- You need source protection — `.love` archives are unencrypted ZIPs.
- You want 3D, or a large first-party ecosystem of built-in systems (physics editors, particle designers, animation state machines).

## Alternatives

- godotengine/godot — full engine with editor, scene system, and 2D+3D when you outgrow code-only.
- defold/defold — Lua-based, ships a real editor and a proven mobile/console distribution pipeline.
- raysan5/raylib — C library with the same minimalist "just draw" philosophy, without the Lua scripting layer.
- pygame/pygame — the Python-ecosystem equivalent: code-only 2D, use when Python matters more than performance.
- Solar2D (coronalabs/Solar2D) — Lua, mobile-first commercial heritage; use when native mobile is the priority.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.1 | 2008 | First public releases (Mercurial/Bitbucket era)[^2]. |
| 0.8.0 | 2012 | Widely-used early stable; broad module coverage. |
| 0.9.0 | 2014 | Per-module API cleanup, threading via Channels. |
| 0.10.0 | 2016 | Touch/mobile improvements, "Super Toast". |
| 11.0 | 2018 | Version jumped 0.10 → 11.0; color range 0–255 → 0–1[^4]. |
| 11.4 | 2022 | Maintenance/stability release on the 11.x line. |
| 11.5 | 2024 | Current stable release. |
| 12.0 | unreleased | SDL3, Vulkan/Metal, graphics API changes on `main`[^3]. |

## References

[^1]: LÖVE wiki, module and callback reference. https://love2d.org/wiki/Main_Page
[^2]: love2d/love repository metadata (GitHub mirror created 2019; project dates to 2008). https://github.com/love2d/love
[^3]: love2d/love README — development on `main`, unstable next-major branch, and the no-AI-contributions policy. https://github.com/love2d/love/blob/main/readme.md
[^4]: LÖVE 11.0 ("Mysterious Mysteries") changelog — color values changed to the 0–1 range. https://love2d.org/wiki/11.0
[^5]: Balatro, a commercially released game built with LÖVE (2024). https://www.playbalatro.com

## Tags

lua, luajit, game-development, 2d-games, game-framework, cpp, gamedev, cross-platform, sdl, love2d
