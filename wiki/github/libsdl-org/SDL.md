# libsdl-org/SDL

> The 25-year-old C library that answers "how do I open a window, read a gamepad, and play audio on every platform" — for games, emulators, and anything else that needs raw multimedia access without an engine.

[GitHub repo](https://github.com/libsdl-org/SDL) ·
[Official website](https://libsdl.org) ·
[License: Zlib](https://github.com/libsdl-org/SDL/blob/main/LICENSE.txt)

## Overview

SDL (Simple DirectMedia Layer) is a cross-platform C library providing windowing, input (keyboard/mouse/gamepad/touch/sensors), audio, haptics, and 2D/GPU rendering behind one API. Sam Lantinga released it in 1998 while porting games to Linux at Loki Software[^1]; he has maintained it ever since, including through his years at Valve. It is the compatibility layer under an enormous share of shipped software: Valve's Steam runtime, Source engine ports, most commercial Linux game ports, and emulators from DOSBox onward. The 16.1k GitHub stars understate reach — the repo only moved to GitHub in 2021 after two decades on libsdl.org's own infrastructure, and most users consume SDL as a system library, not a repo.

SDL's defining tradeoff is that it is a *platform abstraction*, not an engine. There is no scene graph, no asset pipeline, no physics. You get a C API that behaves identically on Windows, macOS, Linux (X11 and Wayland), Android, iOS, and Emscripten, and you build everything else yourself or from the satellite libraries (SDL_image, SDL_ttf, SDL_mixer, SDL_net). That narrow scope is why it has survived every engine generation: engines come and go per project; the "open a window portably" problem is permanent.

The project is in its third major API generation. SDL3 (first stable release 3.2.0, January 2025[^2]) renamed and reorganized nearly the entire API, rewrote the audio subsystem, and added SDL_GPU — a modern graphics abstraction over Vulkan, D3D12, and Metal[^3]. SDL2 remains ubiquitous in the wild; the project ships `sdl2-compat` (SDL2 ABI implemented on top of SDL3) so old binaries keep running[^4]. Development is active — pushes land daily, and 778 open issues reflect triage volume across six-plus platforms, not neglect.

## Getting Started

```bash
# macOS
brew install sdl3
# Debian/Ubuntu (SDL2 is in most distro repos; SDL3 arrived in 2025 releases)
sudo apt install libsdl3-dev
# Windows / any: vcpkg install sdl3, or build from source with CMake
```

Minimal SDL3 program (note: `SDL_Init` returns `bool` in SDL3, and `SDL_CreateWindow` no longer takes x/y coordinates):

```c
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

int main(int argc, char *argv[]) {
    if (!SDL_Init(SDL_INIT_VIDEO)) {
        SDL_Log("SDL_Init failed: %s", SDL_GetError());
        return 1;
    }
    SDL_Window *window = SDL_CreateWindow("hello", 640, 480, 0);
    SDL_Renderer *renderer = SDL_CreateRenderer(window, NULL);

    bool running = true;
    while (running) {
        SDL_Event e;
        while (SDL_PollEvent(&e)) {
            if (e.type == SDL_EVENT_QUIT) running = false;
        }
        SDL_SetRenderDrawColor(renderer, 20, 20, 20, 255);
        SDL_RenderClear(renderer);
        SDL_RenderPresent(renderer);
    }
    SDL_Quit();
    return 0;
}
```

## Architecture / How It Works

SDL is a thin public C API over per-platform backend drivers, selected at runtime. Each subsystem (video, audio, joystick, haptic, camera, sensor) has a driver table; on Linux the video subsystem alone carries X11, Wayland, and KMS/DRM backends and picks one at `SDL_Init` (overridable via `SDL_VIDEO_DRIVER`). This is the core value proposition: the backend matrix is maintained by people who read platform changelogs so you don't.

Rendering comes in three tiers. `SDL_Renderer` is a batched 2D API (textures, rects, geometry) with GPU backends per platform — fine for 2D games and tools, not a general 3D API. SDL_GPU (new in SDL3) is a command-buffer/pipeline abstraction over Vulkan, D3D12, and Metal with its own shader story — a serious mid-level graphics API, though shaders must be provided per-backend or cross-compiled[^3]. Or you bypass both and use SDL purely for window/context creation with your own OpenGL/Vulkan code.

Two internals worth knowing:

- **Dynamic API** — since SDL2, even statically linked apps route every SDL call through a jump table that the `SDL_DYNAMIC_API` environment variable can repoint to a different SDL shared library at runtime[^5]. This is how Steam substitutes updated SDL builds under decade-old game binaries, and why old SDL games keep working on new Wayland desktops.
- **Main-loop inversion** — SDL3 offers optional app callbacks (`SDL_AppInit` / `SDL_AppIterate` / `SDL_AppEvent`) instead of `main()` owning the loop, which is what makes Emscripten and iOS — platforms where the OS owns the loop — first-class targets rather than hacks.

The event queue is the one global chokepoint: all input, window, and device events funnel through a single pumped queue, and on macOS and iOS the video subsystem and event pump must run on the main thread.

## Production Notes

- **SDL2 → SDL3 migration is real work.** Nearly every function was renamed, return conventions changed (int error codes → bool), and coordinates moved from pixels toward points with explicit pixel-density queries. The migration guide is thorough[^6], but budget days, not hours, for a non-trivial codebase — or stay on SDL2, which still receives fixes.
- **Linux backend selection is a support-ticket generator.** Wayland became a preferred video driver in recent releases; apps with X11-era assumptions (global mouse position, window positioning) hit Wayland protocol restrictions that SDL cannot paper over. `SDL_VIDEO_DRIVER=x11` is the classic user-side workaround via XWayland.
- **High-DPI was messy in SDL2** — per-platform flags with inconsistent semantics. SDL3's window-size-in-points model fixed the design; verify on mixed-DPI multi-monitor setups regardless.
- **Gamepad mappings are a database problem.** `SDL_Gamepad` normalizes controllers to an Xbox-style layout using a community mapping database; obscure controllers need mappings shipped with your app (`SDL_AddGamepadMapping`) or they appear as raw joysticks.
- **Audio latency defaults are conservative.** The SDL3 audio rewrite (logical devices, `SDL_AudioStream` with format conversion and resampling) is far more flexible than SDL2's, but low-latency audio still requires explicitly requesting small buffer sizes and checking what you actually got.
- **Threading rules are per-subsystem.** Rendering and event pumping belong on the main thread; audio callbacks fire on an internal thread. The API does not enforce this — it just misbehaves platform-specifically when violated.
- **Bindings lag.** Language bindings (Rust `sdl2`/`sdl3` crates, PySDL2, go-sdl2, C#) trail the C API by months after major releases; check binding maturity before committing to SDL3 from a non-C language.

## When to Use / When Not

**Use when:**
- You are building a game, emulator, or media tool and want engine-free control with portability to desktop + mobile + web from one C codebase.
- You need battle-tested input handling (the gamepad database alone justifies it) or a maintained answer to the Wayland/X11 mess.
- You are writing your own renderer and just need windows, contexts, input, and audio underneath it.

**Avoid when:**
- You want an engine: scenes, assets, physics, editors. SDL gives you none of that; Godot or Unity is the honest choice.
- You only need windowing + input for an OpenGL/Vulkan desktop app and want a smaller dependency — GLFW is leaner if you don't need audio or mobile.
- You need a retained-mode UI toolkit; SDL is the wrong layer (pair it with Dear ImGui, or use Qt/GTK instead).

## Alternatives

- glfw/glfw — use instead when you only need desktop windowing and input for OpenGL/Vulkan and want a much smaller library (no audio, no mobile).
- raysan5/raylib — use instead when you want batteries included (drawing, audio, math, asset loading) for learning or game jams rather than a bare platform layer.
- SFML/SFML — use instead when you want a C++-native object-oriented API covering similar ground for desktop targets.
- floooh/sokol — use instead when you want minimal header-only C libraries for app/gfx/audio and are comfortable assembling the pieces yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 1998 | Initial release by Sam Lantinga at Loki Software[^1]. |
| 1.2 | 2001 | Long-lived stable branch (LGPL); a decade of Linux game ports. |
| 1.2.15 | 2012-01 | Final 1.2 release; branch retired. |
| 2.0.0 | 2013-08 | Full API redesign: multi-window, GameController API; relicensed LGPL → zlib. |
| 2.24.0 | 2022-08 | Version scheme change from 2.0.x to 2.x.y. |
| 3.2.0 | 2025-01 | First stable SDL3: mass API rename, audio rewrite, SDL_GPU, camera API, main callbacks[^2]. |

## References

[^1]: SDL history and credits, libsdl.org. https://www.libsdl.org/credits.php
[^2]: SDL 3.2.0 release — 2025-01. https://github.com/libsdl-org/SDL/releases/tag/release-3.2.0
[^3]: SDL_GPU API documentation. https://wiki.libsdl.org/SDL3/CategoryGPU
[^4]: sdl2-compat — SDL2 ABI on top of SDL3. https://github.com/libsdl-org/sdl2-compat
[^5]: SDL dynamic API documentation. https://github.com/libsdl-org/SDL/blob/main/docs/README-dynapi.md
[^6]: SDL2 to SDL3 migration guide. https://wiki.libsdl.org/SDL3/README-migration

## Tags

c, cross-platform, multimedia, game-development, windowing, input, audio, graphics-abstraction, low-level, gamedev-library
