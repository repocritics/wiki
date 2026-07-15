# SFML/SFML

> A modular C++ multimedia library — windowing, 2D graphics, audio, and networking — that stops deliberately short of being a game engine.

[GitHub repo](https://github.com/SFML/SFML) ·
[Official website](https://www.sfml-dev.org/) ·
[License: Zlib](https://github.com/SFML/SFML/blob/master/license.md)

## Overview

SFML (Simple and Fast Multimedia Library) is a set of C++ modules for building
graphical and multimedia desktop applications, started by Laurent Gomila around
2007[^1]. It gives you a window with an OpenGL context, a 2D drawing API
(sprites, shapes, text, shaders), audio playback and recording, and TCP/UDP
networking — each as a separate library you can link independently. It is one of
the most common first stops for people learning C++ game and graphics
programming.

The defining choice is what SFML is *not*. It is not a game engine: there is no
scene graph, no entity-component system, no physics, no asset pipeline, no
editor. It hands you a `RenderWindow`, a main loop you write yourself, and
primitives to draw into it. This keeps the surface area small and the mental
model transparent, at the cost of everything above "draw a textured quad" being
your responsibility. Teams that want batteries included are usually happier with
a real engine; teams that want to understand every frame they render tend to
stay.

The other long-running tension is scope. SFML targets desktop (Windows, Linux,
macOS) as first-class platforms. Mobile (iOS/Android) support has existed since
the 2.x era but has always been experimental and lightly maintained, and there
is no official web/Emscripten target[^2]. If your roadmap includes phones or the
browser, that constraint should drive the decision early.

## Getting Started

The maintainers recommend the CMake project template, which fetches and builds
SFML alongside your app so you never manage prebuilt binaries[^3]:

```bash
git clone https://github.com/SFML/cmake-sfml-project my-app
cd my-app && cmake -B build && cmake --build build
```

A minimal SFML 3 program (note the C++17 API — scoped enums, `sf::Vector2`
arguments, and `std::optional` event returns):

```cpp
#include <SFML/Graphics.hpp>

int main() {
    sf::RenderWindow window(sf::VideoMode({800, 600}), "SFML");
    sf::CircleShape shape(50.f);
    shape.setFillColor(sf::Color::Green);

    while (window.isOpen()) {
        while (const std::optional event = window.pollEvent()) {
            if (event->is<sf::Event::Closed>())
                window.close();
        }
        window.clear();
        window.draw(shape);
        window.display();
    }
}
```

## Architecture / How It Works

SFML is split into five modules, each a separate static or shared library, with
a strict dependency order:

1. **System** — vectors, clocks, `sf::String`/UTF handling, time. The base
   everything else links. As of 3.0, the old `sf::Thread`/`sf::Mutex`/`sf::Lock`
   wrappers were removed in favor of the C++11 standard library[^4].
2. **Window** — creates the OS window and the OpenGL context, and delivers input
   events. This is the platform-abstraction layer (Win32, X11/Wayland, Cocoa).
3. **Graphics** — the 2D renderer. `Sprite`, `Texture`, `Text`, `Shape`,
   `RenderTexture`, and GLSL `Shader` all funnel into OpenGL draw calls under the
   hood. This is 2D only; there is no 3D scene support.
4. **Audio** — playback, streaming, and capture. In 3.0 the backend was switched
   from OpenAL-Soft to the bundled miniaudio library[^4], visible in the
   dependency list (miniaudio, dr_mp3) rather than a linked OpenAL runtime.
5. **Network** — TCP/UDP sockets, a `Packet` serialization helper, and small
   HTTP/FTP clients.

Text rendering pulls in FreeType for glyph rasterization and, since 3.0,
HarfBuzz and SheenBidi for shaping and bidirectional layout. Image loading uses
stb_image; audio codecs use libogg/vorbis/flac plus miniaudio. All of these are
vendored, so a default build has no external system dependencies beyond the OS
graphics/audio stack.

Because Graphics is a thin layer over OpenGL, you can freely interleave raw
OpenGL calls with SFML drawing (`pushGLStates`/`popGLStates` guard the boundary).
The OpenGL context is thread-affine: rendering must happen on the thread that
owns the context, and moving the context between threads requires explicit
`setActive` calls. This is the source of most "nothing draws" bugs in
multithreaded SFML code.

## Production Notes

**SFML 3.0 is a hard break from 2.x.** Released at the end of 2024, it requires
C++17, replaces most integer enums with scoped enums, passes `sf::Vector2`
instead of separate x/y arguments in many APIs, returns `std::optional` from
`pollEvent`, and removes deprecated 2.x surface[^4]. There is an official
migration guide, but expect real work — this is not a drop-in upgrade, and
third-party tutorials written for 2.x will not compile.

**2.x is feature-frozen.** The README states plainly that no more features are
planned for the 2.x series; development is on `master` (3.x)[^5]. New projects
should start on 3.0; existing 2.x projects get bug fixes only.

**It is not an engine, and that shows up as missing infrastructure.** No physics
(pair it with Box2D or Chipmunk), no built-in ECS, no tilemap or animation
system, no packaging/asset story. The main loop, fixed-timestep logic, resource
management, and scene handling are all yours to write. This is fine for learning
and small games and painful for large ones.

**Platform reality.** Desktop is solid and well-tested. iOS/Android support is
experimental and historically lags behind desktop; do not assume mobile parity.
There is no official Emscripten/WebAssembly target, though community ports
exist. If you need broad platform reach, SDL is the safer base.

**Performance.** The Graphics module batches poorly by default — each `draw`
call is roughly one OpenGL draw. Rendering thousands of independent sprites
naively is slow; the standard fix is to combine them into a single
`sf::VertexArray` (or vertex buffer) so they draw in one call. This is a common
surprise for people expecting an automatically batched renderer.

**Bindings drift.** SFML has a C binding (CSFML) and third-party language
bindings (rust-sfml, SFML.Net, and others). These track upstream at varying
speeds; several older bindings still target 2.x and have not been updated for 3.

## When to Use / When Not

**Use when:**
- You are learning C++ graphics/game programming and want a small, readable API.
- You are building a 2D game, prototype, or multimedia tool for desktop.
- You want windowing + 2D drawing + audio + networking from one coherent library
  without adopting a full engine.
- You value being able to drop down to raw OpenGL when you need to.

**Avoid when:**
- You need 3D — SFML has no 3D scene support; use an engine or a lower-level GL/Vulkan stack.
- Mobile or web are primary targets — support is experimental or absent.
- You want batteries included (physics, ECS, editor, asset pipeline).
- You need to render very large numbers of dynamic objects without hand-writing
  batching.

## Alternatives

- libsdl-org/SDL — lower-level C library with much broader platform reach
  (including mobile and, via ports, web); use it when platform coverage or a C
  API matters more than a built-in 2D drawing abstraction.
- raysan5/raylib — simpler C API with 3D, models, and more built-in helpers; use
  it when you want fewer moving parts and don't need SFML's networking module.
- glfw/glfw — windowing, input, and GL/Vulkan context only; use it when you will
  write your own renderer and want nothing else in the way.
- godotengine/godot — a full 2D/3D engine with editor and scene system; use it
  when you want batteries included rather than a library.
- liballeg/allegro5 — comparable-scope cross-platform game library; use it if you
  prefer its API or C interface.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2007 | First public release by Laurent Gomila[^1]. |
| 1.6 | 2010 | Final release of the 1.x line. |
| 2.0 | 2013 | Major rewrite: new API, `RenderTexture`, GLSL shaders, vertex arrays. |
| 2.6.0 | 2023 | Last feature release of the 2.x series[^5]. |
| 3.0.0 | 2024-12 | C++17 required; scoped enums, `sf::Vector2` APIs, `std::optional` events; audio backend moved OpenAL → miniaudio[^4]. |

## References

[^1]: SFML history and origins, SFML website. https://www.sfml-dev.org/
[^2]: SFML tutorials (per-platform, desktop-focused). https://www.sfml-dev.org/tutorials/
[^3]: SFML CMake project template. https://github.com/SFML/cmake-sfml-project
[^4]: SFML 3.0 migration guide and changelog. https://github.com/SFML/SFML/blob/master/changelog.md
[^5]: SFML/SFML README — "No more features are planned for the 2.x release series." https://github.com/SFML/SFML

## Tags

cpp, multimedia, 2d-graphics, opengl, audio, game-development, cross-platform, windowing, networking, library
