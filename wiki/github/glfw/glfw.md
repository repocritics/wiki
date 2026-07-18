# glfw/glfw

> The minimal C windowing and input layer for OpenGL and Vulkan on the desktop
> — it opens a window, creates a context, reads input, and deliberately does
> nothing else.

[GitHub repo](https://github.com/glfw/glfw) ·
[Official website](https://www.glfw.org) ·
[License: Zlib](https://www.glfw.org/license.html)

## Overview

GLFW is a multi-platform library for creating windows, OpenGL/OpenGL ES
contexts, and Vulkan surfaces, and for reading keyboard, mouse, and gamepad
input. It started in 2002 as Marcus Geelnard's replacement for the aging GLUT
toolkit; Camilla Löwy (elmindreda) has led the project since the 2.x era and
authored the ground-up 3.0 rewrite in 2013[^1]. It is written in C99 (with
Objective-C for the macOS backend), supports Windows, macOS, and Linux/Unix
(both X11 and Wayland), and is licensed under Zlib — permissive enough for
static linking into closed-source games without attribution ceremony.

Its defining trait is scope discipline. GLFW has no audio, no image loading,
no font rendering, no 2D drawing, and no mobile or console support. That makes
it the standard bottom layer under hand-rolled OpenGL/Vulkan renderers, engine
prototypes, and tooling (the Dear ImGui GLFW backend is a near-universal
pairing), and it is what most language bindings wrap: LWJGL for Java,
go-gl/glfw for Go, pyGLFW for Python. The tradeoff: everything outside
"window + context + input" is your problem, and the moment you need audio or
Android you are assembling libraries or switching to SDL.

At 15.2k stars and 5.9k forks it is one of the most-depended-on C libraries in
graphics. Maintenance is real but slow-cadence: five years passed between 3.3
(2019) and 3.4 (2024), master still receives regular fixes (last push June
2026), and the ~760 open issues reflect a tiny maintainer team triaging a very
large user base rather than abandonment.

## Getting Started

```bash
# macOS               # Debian/Ubuntu                  # Windows
brew install glfw     sudo apt install libglfw3-dev    vcpkg install glfw3
```

```c
// main.c — cc main.c -lglfw -framework OpenGL (macOS) / -lGL (Linux)
#include <GLFW/glfw3.h>

int main(void) {
    if (!glfwInit()) return -1;
    GLFWwindow* window = glfwCreateWindow(640, 480, "Hello", NULL, NULL);
    if (!window) { glfwTerminate(); return -1; }
    glfwMakeContextCurrent(window);
    glfwSwapInterval(1);                      // vsync
    while (!glfwWindowShouldClose(window)) {
        glClear(GL_COLOR_BUFFER_BIT);
        glfwSwapBuffers(window);
        glfwPollEvents();
    }
    glfwTerminate();
    return 0;
}
```

With CMake: `find_package(glfw3 REQUIRED)` + `target_link_libraries(app glfw)`,
or vendor the source via `add_subdirectory(glfw)`. GLFW does not load modern
OpenGL function pointers — pair it with a loader such as glad, passing
`glfwGetProcAddress` as the resolver.

## Architecture / How It Works

GLFW is one library with per-platform backends behind a common internal
interface: Win32, Cocoa, X11, Wayland, and a headless "null" platform. Context
creation is likewise abstracted over WGL, NSGL, GLX, EGL, and OSMesa. Since
3.4, backends are selected at runtime rather than compile time: `glfwInit`
probes available platforms, and on Linux prefers Wayland over X11 when a
compositor is present — overridable via the `GLFW_PLATFORM` init hint[^2].

The event model is callback-based and main-thread-bound. You register callbacks
(key, cursor, framebuffer resize, drop, etc.) and pump the queue with
`glfwPollEvents` or `glfwWaitEvents`; most window and event functions must be
called from the main thread, a constraint inherited from Cocoa and Win32 rather
than a GLFW design choice[^3]. Context calls (`glfwMakeContextCurrent`,
`glfwSwapBuffers`) may run on any thread, which is what makes the standard
"render thread + main-thread event pump" architecture possible.

For Vulkan, GLFW creates no context at all: it reports the required instance
extensions and creates a `VkSurfaceKHR` via `glfwCreateWindowSurface`;
everything else is raw Vulkan[^4]. Gamepad support maps heterogeneous joystick
hardware onto an Xbox-style layout using the community SDL_GameControllerDB
database[^5]. On Linux, GLFW loads system libraries (libGL, libwayland-client,
etc.) dynamically at runtime, keeping link-time dependencies minimal.

## Production Notes

- **The Windows modal-loop freeze.** During interactive resize or drag, Win32
  traps the thread in a modal loop and `glfwPollEvents` does not return, so
  single-threaded render loops visibly freeze. The most-reported GLFW footgun;
  the fix is rendering from a second thread or the window refresh callback.
- **Wayland is not X11.** Under Wayland there is no global window positioning
  (`glfwSetWindowPos`/`glfwGetWindowPos` do nothing), no programmatic
  raise/focus, and GNOME shows crude fallback decorations unless libdecor is
  installed. Apps assuming X11 semantics break quietly; many shipped titles
  pin `GLFW_PLATFORM` to X11 (via XWayland) instead.
- **HiDPI is per-platform work.** Framebuffer size and window size differ
  under scaling; use `glfwGetFramebufferSize` for `glViewport`, never the
  window size, and handle content-scale callbacks for monitor changes.
- **Version pinning matters more than usual.** With five years between 3.3
  and 3.4, distro packages diverge widely, and 3.4's Linux platform-selection
  change surfaced Wayland bugs in apps only ever tested on X11. Vendoring the
  source is the common defense.
- **Scope gaps you will hit.** No audio, networking, or image decoding, no
  IME-heavy text input story comparable to SDL's, no mobile/console backends.
  Budget for companion libraries (miniaudio, stb, glad) from day one.

## When to Use / When Not

**Use when:**
- You are writing your own OpenGL or Vulkan renderer and want the thinnest
  possible portable window/input layer.
- You need a permissive-license (Zlib) dependency that statically links into
  proprietary desktop software.
- You are building tools or engine prototypes around Dear ImGui.
- You want a stable C ABI that language bindings wrap cleanly.

**Avoid when:**
- You target mobile or consoles — GLFW is desktop-only; SDL covers far more
  platforms.
- You need audio, 2D rendering, or robust IME/text input out of the box.
- You want batteries included for learning or game jams — raylib gets you to
  pixels faster.
- You are in Rust — winit is the ecosystem-native equivalent with better
  integration into wgpu.

## Alternatives

- libsdl-org/SDL — use instead when you need audio, 2D rendering, haptics, or
  platforms beyond the desktop; much larger API surface, same C portability.
- rust-windowing/winit — use instead in Rust; the windowing layer wgpu and
  most Rust graphics crates assume.
- raysan5/raylib — use instead for teaching, prototypes, and jams; includes
  drawing, audio, and asset loading at the cost of engine-like opinions.
- floooh/sokol — use instead for single-header minimalism where sokol_app +
  sokol_gfx replace GLFW + a GL loader with one coherent cross-API layer.
- freeglut — use only to keep legacy GLUT-based coursework or samples alive.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2002 | Initial release by Marcus Geelnard (SourceForge era)[^1]. |
| 3.0 | 2013-06 | Full API rewrite: multi-window, multi-monitor, error callbacks; broke 2.x compat[^6]. |
| 3.1 | 2015-01 | Custom cursors, file drop, character-with-modifiers callback. |
| 3.2 | 2016-06 | Vulkan surface support, window icons, `glfwSetWindowMonitor`. |
| 3.3 | 2019-04 | Gamepad mappings (SDL_GameControllerDB), content scale, raw mouse motion. |
| 3.4 | 2024-02 | Runtime platform selection, Wayland preferred on Linux, mouse passthrough, ANGLE backend hint[^2]. |

## References

[^1]: GLFW FAQ — project origins and maintainership. https://www.glfw.org/faq.html
[^2]: GLFW release notes for 3.4. https://www.glfw.org/docs/latest/news.html
[^3]: GLFW intro guide — thread safety constraints. https://www.glfw.org/docs/latest/intro_guide.html#thread_safety
[^4]: GLFW Vulkan guide. https://www.glfw.org/docs/latest/vulkan_guide.html
[^5]: SDL_GameControllerDB — community gamepad mapping database. https://github.com/mdqinc/SDL_GameControllerDB
[^6]: GLFW version history / changelog. https://www.glfw.org/changelog.html

## Tags

c, opengl, vulkan, windowing, input-handling, cross-platform, graphics, game-development, desktop, native-library
