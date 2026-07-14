# raysan5/raylib

> A single-C-file-family game programming library for people who want to draw pixels without an engine, an editor, or a dependency tree.

[GitHub repo](https://github.com/raysan5/raylib) ·
[Official website](http://www.raylib.com) ·
[License: Zlib](https://github.com/raysan5/raylib/blob/master/LICENSE)

## Overview

raylib is a C99 library for videogame and graphics programming, started by Ramon Santamaria (raysan5) in 2013 and inspired explicitly by the Borland BGI graphics library and Microsoft's XNA framework[^1]. It is not a game engine: there is no scene graph, no editor, no asset pipeline, no entity system. You get an immediate-mode drawing API (`BeginDrawing()` / `EndDrawing()`), window and input handling, audio, math, and model loading, and you write the game loop yourself. The stated audience is prototyping, tooling, education, and embedded/graphical applications, and the README is unusually blunt about this "spartan" posture[^1].

The defining design decision is **zero external dependencies**: every third-party library raylib needs (GLFW-derived windowing, stb image/font loaders, miniaudio, etc.) is vendored into `src/external`, so a raylib build pulls nothing from the network and links against nothing you did not compile[^2]. This is why raylib packages cleanly on dozens of OS distributions and why it is a common first-C-graphics library. The tension is the flip side of the same coin: because raylib owns its whole stack and exposes a flat C API, it is easy to start and hard to extend past what the API anticipates — there is no plugin surface, and non-trivial projects end up either dropping to the `rlgl` layer or forking.

raylib's other notable trait is reach. The core is plain C with PascalCase/camelCase naming, which makes it trivial to bind: the project tracks bindings to 70+ languages[^3], and the same source targets Windows, Linux, macOS, Raspberry Pi, Android, and the web (via Emscripten/WebAssembly), plus experimental RISC-V and ESP32 embedded targets.

## Getting Started

raylib ships prebuilt binaries on the GitHub Releases page and is available through most package managers (vcpkg, Homebrew, apt, pacman, Conan). From source with CMake:

```bash
git clone https://github.com/raysan5/raylib
cd raylib
cmake -B build -DBUILD_EXAMPLES=OFF
cmake --build build
```

A minimal program — note the immediate-mode loop, redrawn every frame:

```c
#include "raylib.h"

int main(void)
{
    InitWindow(800, 450, "raylib example - basic window");
    SetTargetFPS(60);

    while (!WindowShouldClose())   // Esc or window close
    {
        BeginDrawing();
            ClearBackground(RAYWHITE);
            DrawText("Congrats! You created your first window!", 190, 200, 20, LIGHTGRAY);
        EndDrawing();
    }

    CloseWindow();
    return 0;
}
```

Link against `raylib` plus its platform libs (`-lraylib -lGL -lm -lpthread -ldl -lrt -lX11` on Linux). There is no generated API reference; the learning path is the 140+ examples and the single-page cheatsheet[^4].

## Architecture / How It Works

raylib is a layered set of header/implementation `.c` files, each usable in near-isolation:

- **raylib core** — window, input, timing, and the immediate-mode `Draw*` API. Everything above the GPU.
- **rlgl** (`rlgl.h`) — a single-header OpenGL abstraction layer that normalizes OpenGL 1.1, 2.1, 3.3, 4.3, ES 2.0, and ES 3.0 behind a small immediate-mode-style API. rlgl is designed to be pulled out and used standalone; it is where the version-specific GL code lives, which is why raylib can target a Raspberry Pi and a desktop from one codebase[^2].
- **raymath** (`raymath.h`) — a header-only vector/matrix/quaternion math module, also usable independently.
- **rlsw** — a software renderer backend that lets raylib draw without any OpenGL context at all, added for headless and no-GPU targets.
- Vendored loaders in `src/external` — image (stb_image), TTF/OTF/BDF fonts, audio decoders (WAV/QOA/OGG/MP3/FLAC/XM/MOD), and model formats (glTF, IQM, M3D, OBJ).

The platform abstraction is the load-bearing internal detail. Desktop builds default to a GLFW-derived backend, but raylib compiles the same core against several `rcore_*` platform modules (desktop GLFW, SDL, DRM/native, web, Android). Choosing a backend is a compile-time `PLATFORM_*` decision, not a runtime one.

Because the API is immediate-mode and stateless-looking, there is no retained scene: you re-issue every draw call every frame. This keeps the mental model tiny but means all higher-level structure (culling, batching decisions, ECS, UI state) is your responsibility. Batching does happen internally — rlgl accumulates vertices and flushes on state changes or buffer limits — but you feel it only as a performance characteristic, not an API.

## Production Notes

**It is a library, not an engine — plan for what it omits.** No level editor, no built-in physics (use a separate lib like Chipmunk/Box2D or physac), no GUI toolkit in core (raygui is a separate sibling repo), no asset hot-reload, no networking. Teams that outgrow prototyping often build these themselves; that work is the real cost of choosing raylib for a shipping title.

**Immediate mode has a per-frame cost model.** Drawing thousands of individual `DrawText`/`DrawRectangle` calls issues many small draw operations; performance comes from letting rlgl batch same-texture, same-shader geometry. Frequent texture/shader switches break the batch. For sprite-heavy scenes, atlas your textures and group draws by material.

**Web builds are their own project.** Emscripten targets cannot use a blocking `while (!WindowShouldClose())` loop — the browser owns the event loop, so you must restructure around `emscripten_set_main_loop`. File I/O, threads, and audio all behave differently under WASM. This is the single most common source of "works on desktop, breaks on web" reports.

**API stability across major versions.** raylib is not shy about breaking changes between majors — function signatures, struct layouts, and enum values have changed (for example the 4.x line reworked parts of the camera, audio, and text APIs, and 5.0 continued function-level renames). Pin to a specific tag, read the release notes before upgrading, and expect to touch call sites. This is manageable because the surface is small, but it is not a "bump the version" ecosystem.

**Single-maintainer project.** raysan5 is the dominant author and gatekeeper. That yields a coherent, opinionated API and fast iteration, but it also means the roadmap and merge decisions track one person's taste; large architectural PRs that don't fit the "keep it simple" philosophy are routinely declined. Treat the vendored `src/external` libs as pinned — updating them is a raylib-side decision, not something you patch downstream.

**Bindings lag core.** The 70+ language bindings are community-maintained and version-skew against the C core; a raylib-5.x feature may not be exposed in your language's binding yet. Check the binding's own version before assuming parity.

## When to Use / When Not

**Use when:**
- You're learning graphics/game programming in C or a bound language and want to draw something in ten lines.
- You're building a prototype, tool, visualizer, or game jam entry and want no build friction and no dependency management.
- You need one small codebase to run on desktop, web, Raspberry Pi, and mobile.
- You want to embed a lightweight renderer in a larger C/C++ app without adopting an engine.

**Avoid when:**
- You need an editor, scene system, physics, and asset pipeline out of the box — that's an engine's job.
- You're shipping a large team title where retained-mode tooling and a component ecosystem save more time than raylib's simplicity gives back.
- You require long-term API stability with rare breaking changes across majors.
- You need first-class 3D/PBR authoring workflows; raylib loads models and does basic PBR, but it is not a 3D content pipeline.

## Alternatives

- libsdl-org/SDL — lower-level cross-platform windowing/input/audio; use when you want the substrate raylib is built near and will assemble rendering yourself.
- floooh/sokol — even more minimal single-header C libs (gfx/app/audio); use when you want a modern explicit GPU abstraction and to pick your own scope.
- godotengine/godot — full open-source engine with editor and scene system; use when you've outgrown a library and want batteries included.
- glfw/glfw — just windowing + OpenGL/Vulkan context + input; use when raylib's drawing API is more than you want and you'll bring your own renderer.
- love2d/love — batteries-included 2D framework scripted in Lua; use when you want raylib's simplicity but prefer a scripting language and a 2D focus.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013-11 | Initial release; created for a game programming course[^1]. |
| 2.0 | 2018-07 | No external dependencies milestone; self-contained build. |
| 3.0 | 2020-04 | rcore/rlgl restructure, file-system and data abstraction. |
| 4.0 | 2021-11 | rcore platform split groundwork, API cleanups. |
| 4.2 | 2022-08 | Expanded platform/audio/format support. |
| 5.0 | 2024-01 | Platform backend modularization (`rcore_*`), API renames. |
| 6.0 | 2026 | Latest major line; continued backend and format work[^5]. |

## References

[^1]: raylib README and project history — origin as a course library, BGI/XNA inspiration, "spartan" design statement. https://github.com/raysan5/raylib and https://www.raylib.com
[^2]: raylib architecture notes — rlgl abstraction layer, vendored dependencies, platform backends. https://github.com/raysan5/raylib/wiki/raylib-architecture
[^3]: raylib language bindings list (70+ languages). https://github.com/raysan5/raylib/blob/master/BINDINGS.md
[^4]: raylib cheatsheet and examples collection (primary docs; no generated API reference). https://www.raylib.com/cheatsheet/cheatsheet.html
[^5]: raylib GitHub Releases — version tags and changelogs. https://github.com/raysan5/raylib/releases

## Tags

c, game-development, graphics, opengl, immediate-mode, cross-platform, webassembly, embedded, gamedev, library, education
