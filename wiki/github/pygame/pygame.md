# pygame/pygame

> A Python wrapper around SDL for 2D games and multimedia — the default "first game library" for Python, now shadowed by its own community fork.

[GitHub repo](https://github.com/pygame/pygame) ·
[Official website](https://www.pygame.org) ·
License: LGPL-2.1-only

## Overview

pygame is a Python library for writing 2D games and multimedia applications on top of SDL (Simple DirectMedia Layer)[^1]. It wraps SDL's windowing, input, image, audio, and font subsystems in a Python API centered on two objects: `Surface` (a blittable pixel buffer) and `Rect` (an integer rectangle used for positioning and collision). The project dates to 2000, started by Pete Shinners; the GitHub repository here was created in 2017 when development moved off Bitbucket/Mercurial. It is one of the most widely taught game libraries in Python, heavily used in classrooms, game jams, and hobby projects.

The defining fact about this repo in 2026 is governance, not code. In 2023 a large share of the active contributor base forked into **pygame-ce** ("pygame Community Edition") after a dispute with the project's owner over control and the pygame trademark[^2]. pygame-ce publishes on PyPI as `pygame-ce` but still imports as `import pygame`, so it is a drop-in replacement, and it now ships releases and new features faster than this upstream repo. Anyone starting a project in 2026 should make a deliberate choice between the two; most active tutorials and community answers increasingly assume pygame-ce.

The underlying tension is that pygame is a thin, mature wrapper over a C library. That makes it stable, portable, and easy to learn, but also means it inherits SDL's software-blitting model: the classic `Surface` API is CPU-bound, single-threaded against the Python GIL, and not GPU-accelerated. It is excellent for learning and for games that fit within a software-rendered 2D budget, and a poor fit when you need shaders, many thousands of sprites, or 3D.

## Getting Started

```bash
pip install pygame          # upstream (this repo)
# or:
pip install pygame-ce       # community fork; also imports as `pygame`
```

```python
import pygame

pygame.init()
screen = pygame.display.set_mode((640, 480))
clock = pygame.time.Clock()

running = True
while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    screen.fill((30, 30, 30))            # clear
    pygame.draw.circle(screen, (255, 80, 80), (320, 240), 40)
    pygame.display.flip()                # present the frame
    clock.tick(60)                       # cap the loop at ~60 FPS

pygame.quit()
```

`python -m pygame.examples.aliens` runs a bundled sample game to verify the install.

## Architecture / How It Works

pygame is a set of C extension modules (built against CPython's C API) plus a thin Python layer. Each module maps to an SDL subsystem or a vendored companion library:

- **`display` / `Surface`** — SDL video. A `Surface` is a pixel buffer; `blit` copies one surface onto another. The classic path is software rendering — the CPU composites everything, then one final surface is presented.
- **`draw`, `transform`, `image`** — primitive drawing, scaling/rotation (`transform` embeds SDL_rotozoom), and image loading via SDL_image (libpng/libjpeg).
- **`font`** — text via SDL_ttf → FreeType. **`freetype`** is a separate, richer text module.
- **`mixer` / `mixer.music`** — audio via SDL_mixer (WAV/OGG/MP3).
- **`sprite`** — a pure-Python convenience layer: `Sprite`, `Group`, and group-based collision helpers. Not part of SDL; it is optional sugar over `Surface` + `Rect`.
- **`surfarray`** — zero-copy views of surface pixels as NumPy arrays for per-pixel work. NumPy is an optional dependency.

pygame 2 (2020) rebased the whole library from SDL 1.2 onto **SDL2**[^3]. SDL2 brought better multi-monitor, hardware-backed windows, and touch/gamepad support. pygame 2 also exposes `pygame._sdl2.video`, an experimental `Texture`/`Renderer` API that does use GPU acceleration — but it is a separate, less-documented path from the mainstream `Surface` API, and most tutorials and third-party code still target software surfaces.

The event model is a single global queue drained with `pygame.event.get()` inside the main loop. There is no scene graph, no entity system, no asset pipeline, and no built-in physics — pygame gives you a window, a pixel buffer, input, and sound, and leaves architecture to the developer. This minimalism is why it is easy to teach and why non-trivial games end up reinventing the same subsystems.

## Production Notes

**Upstream vs. pygame-ce is the first decision.** They share a namespace and most of an API, so `import pygame` works either way, but versions diverge: pygame-ce has shipped features (e.g. improved `SRCALPHA` handling, new `Window`/`Renderer` conveniences, more `_sdl2` coverage) ahead of upstream. Pin exactly one in your dependencies — installing both into the same environment clobbers the `pygame` import and produces confusing breakage.

**Performance is CPU-bound by default.** Software blitting plus the GIL means frame budgets get eaten by pixel work in Python. Practical mitigations: pre-`convert()`/`convert_alpha()` every surface once at load time (huge blit speedup); minimize per-frame allocations; batch pixel math through `surfarray` + NumPy instead of `set_at`/`get_at` loops; cap draw calls. When you genuinely need thousands of sprites or effects, move to the `_sdl2` Renderer/Texture path or a GPU-first library.

**`convert()` before a display exists throws.** Surfaces can only be converted after `set_mode()` has created a video context. A common beginner bug is loading and converting images at module import time.

**Distribution is the hard part.** pygame apps are Python apps: shipping to non-developers means bundling the interpreter and SDL shared libraries with PyInstaller, Nuitka, or briefcase. Web deployment is possible via **pygbag** (Pyodide/WASM), but with real limits on threads, blocking I/O, and load size. There is no first-party mobile or console story.

**Version/platform friction.** Wheels lag new Python releases at times, so a fresh CPython version can briefly have no matching pygame wheel, forcing a source build (and an SDL toolchain). The MP3 support in `mixer` depends on how SDL_mixer was compiled and can vary by platform. Prefer OGG for portability.

## When to Use / When Not

**Use when:**
- You are learning game programming, teaching, or doing a game jam and want minimal ceremony.
- You need a small, portable 2D game or multimedia tool (visualizers, kiosks, simple simulations).
- You want direct pixel control and are comfortable building your own game architecture.
- You want a stable, long-lived API with an enormous body of tutorials and StackOverflow answers.

**Avoid when:**
- You need GPU shaders, lighting, or high sprite counts — the default renderer is software.
- You need 3D — pygame is 2D-only (use a real engine).
- You need to ship to mobile, consoles, or the web as a first-class target.
- You want batteries-included scenes/physics/tweening/UI — pygame ships none of that.

## Alternatives

- pygame-community/pygame-ce — the actively-developed community fork; drop-in `import pygame`. Use instead of upstream when you want faster releases and newer features.
- pythonarcade/arcade — OpenGL-backed 2D with a modern, higher-level API. Use when you need hardware acceleration and built-in helpers (cameras, sprite lists, tilemaps).
- pyglet/pyglet — pure-Python, OpenGL windowing with no SDL/C-extension dependency. Use when you want GPU rendering and a lighter install footprint.
- kitao/pyxel — opinionated retro/pixel-art engine with tight constraints. Use when you want a fixed 16-color, chiptune aesthetic and built-in tools.
- panda3d/panda3d — full 3D engine with a Python API. Use when the project is actually 3D or needs a real engine's scene graph and asset pipeline.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2001 | Initial release by Pete Shinners, built on SDL 1.2[^1]. |
| 1.9.x | 2009–2019 | Long-lived stable series; Python 2 and early Python 3. |
| (repo move) | 2017-03-26 | Development migrated to this GitHub repository. |
| 2.0.0 | 2020-10 | Rebased onto SDL2; Python 3-focused[^3]. |
| 2.1–2.6.x | 2021–2025 | Incremental SDL2 releases; `_sdl2` renderer expands. |
| pygame-ce fork | 2023 | Community fork over governance/trademark dispute[^2]. |

## References

[^1]: pygame README and project history; original author Pete Shinners. https://www.pygame.org/wiki/about
[^2]: pygame Community Edition — project and rationale for the fork. https://pyga.me/ and PyPI `pygame-ce`. https://pypi.org/project/pygame-ce/
[^3]: pygame 2.0.0 release notes (SDL2 migration). https://www.pygame.org/news.html

## Tags

python, game-development, 2d-games, sdl, sdl2, multimedia, c-extension, game-library, graphics, cross-platform
