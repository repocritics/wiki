# rust-windowing/winit

> Cross-platform window creation and input event handling in pure Rust — and nothing else.

[GitHub repo](https://github.com/rust-windowing/winit) ·
[Documentation](https://docs.rs/winit/) ·
[License: Apache-2.0](https://github.com/rust-windowing/winit/blob/master/LICENSE)

## Overview

winit is the de facto windowing layer for the Rust graphics ecosystem. It opens
OS windows, drives an event loop, and normalizes input (keyboard, mouse, touch,
IME, DPI/scale-factor changes) across Windows, macOS, Linux (X11 and Wayland),
iOS, Android, Web (WASM/canvas), and Redox OS[^1]. The scope is deliberately
narrow: winit creates the surface and delivers events; it does **not** render
anything, draw widgets, manage OpenGL/Vulkan contexts, or provide a GUI toolkit.
The README states this plainly — it is "a low-level brick in a hierarchy of
libraries"[^1].

That narrowness is the whole design tension. To put a pixel on screen you must
hand winit's window to a separate renderer, and the bridge is the
`raw-window-handle` crate: winit exposes a `RawWindowHandle`/`RawDisplayHandle`
that wgpu, glutin (OpenGL), or softbuffer consume to create a drawing surface.
Almost nobody uses winit directly; they use it because Bevy, egui/eframe, iced,
wgpu's examples, and most Rust game/GUI projects sit on top of it. A breaking
winit release therefore ripples through every downstream framework. The project
has never reached 1.0 and has shipped several full API rewrites, so pinning an
exact version and upgrading deliberately is standard practice, not caution.

## Getting Started

```toml
[dependencies]
winit = "0.30"        # 0.31 is in beta as of mid-2026
```

A minimal application on the current (`ApplicationHandler`) API. This opens a
window and exits on close; you supply your own renderer in `RedrawRequested`:

```rust
use winit::application::ApplicationHandler;
use winit::event::WindowEvent;
use winit::event_loop::{ActiveEventLoop, ControlFlow, EventLoop};
use winit::window::{Window, WindowId};

#[derive(Default)]
struct App { window: Option<Window> }

impl ApplicationHandler for App {
    fn resumed(&mut self, event_loop: &ActiveEventLoop) {
        let attrs = Window::default_attributes().with_title("winit");
        self.window = Some(event_loop.create_window(attrs).unwrap());
    }

    fn window_event(&mut self, event_loop: &ActiveEventLoop, _id: WindowId, event: WindowEvent) {
        match event {
            WindowEvent::CloseRequested => event_loop.exit(),
            WindowEvent::RedrawRequested => {
                // draw here, then schedule the next frame:
                self.window.as_ref().unwrap().request_redraw();
            }
            _ => {}
        }
    }
}

fn main() {
    let event_loop = EventLoop::new().unwrap();
    event_loop.set_control_flow(ControlFlow::Poll); // Poll for games, Wait for UIs
    event_loop.run_app(&mut App::default()).unwrap();
}
```

## Architecture / How It Works

winit is a facade over per-platform backends selected at compile time: Win32 on
Windows, AppKit on macOS, UIKit on iOS, a Wayland client and an Xlib/XCB client
on Linux, `android-activity` on Android, and DOM/canvas bindings on Web[^1]. A
single `Event`/`WindowEvent` vocabulary is projected onto all of them, which
means winit is a lowest-common-denominator abstraction — a feature only exists
in the cross-platform API if it can be expressed on (most of) the backends. The
`winit::platform::*` modules hold the escape hatches for per-OS behavior.

The **event loop** is the core. Since 0.30 the model is the `ApplicationHandler`
trait: you implement methods (`resumed`, `window_event`, `device_event`,
`about_to_wait`, `suspended`) and call `EventLoop::run_app`[^2]. This replaced
the older closure-based `EventLoop::run(|event, elwt| ...)` design. `ControlFlow`
picks the loop's blocking behavior — `Poll` runs continuously (games),
`Wait` sleeps until the next event (desktop UIs).

The `resumed`/`suspended` split is not busywork — it exists because of
**Android**, where the OS can destroy and recreate the native window under a
running process. The correct pattern is to create your GPU surface in `resumed`
and drop it in `suspended`; code written only against a desktop mental model
breaks on mobile precisely here.

**DPI** lives in a separate `dpi` crate (vendored here, with its own license
note[^1]): `PhysicalSize`/`LogicalSize` plus an explicit scale factor and
`ScaleFactorChanged` events when a window crosses monitors. **Input** uses a
`KeyEvent` model separating physical key location from logical (layout-dependent)
key and text, replacing the older `VirtualKeyCode` enum.

## Production Notes

**winit renders nothing — you must pair it.** The near-universal companions are
wgpu (Vulkan/Metal/DX12/WebGPU), glutin (OpenGL contexts), and softbuffer (CPU
pixel buffer). All three interoperate through `raw-window-handle`; version
mismatches in that crate between winit and your renderer are a classic build
failure, because a `raw-window-handle` major bump is a breaking change across
the whole chain.

**API churn is the dominant upgrade cost.** winit has no stability guarantee
(0.x) and has done several rewrites. The 0.30 migration to `ApplicationHandler`
was mechanical but touched every app's entry point; downstream frameworks (Bevy,
eframe, iced) each absorbed it on their own schedule, so a winit bump often can't
happen until your framework ships a compatible release. Do not float winit;
pin it.

**Wayland vs X11 differ at runtime on the same Linux binary.** winit picks a
backend at startup. Window decorations on Wayland are client-side — winit draws
its own titlebar via SCTK unless the compositor offers server-side decorations,
so decoration/appearance behavior legitimately varies between the two.

**Web has no blocking loop.** In the browser you cannot block the main thread, so
the run model is spawn-based and control never "returns". Android likewise needs
the `android-activity` glue and an `android_main` entry point, not a normal
`main`. **MSRV is 1.85** (raised via minor bumps); Android may require a newer
toolchain and Redox requires nightly[^1]. The ~650 open issues are expected for a
project mediating this many OS backends — it is actively developed, with weekly
maintainer meetings[^1], not abandoned.

## When to Use / When Not

**Use when:**
- You need one windowing/input abstraction spanning desktop, mobile, and web.
- You're building or embedding a renderer (wgpu/glutin/softbuffer) and want the OS glue handled.
- You're writing a game, GUI toolkit, or graphics tool and want the ecosystem-standard base that Bevy/egui/iced already assume.

**Avoid when:**
- You want batteries included (audio, gamepad, rendering, native menus) — winit is windowing only.
- You need native menu bars or a system tray — winit omits these; see the tao fork.
- You target a single OS and a C library (SDL, GLFW) is simpler for your case.
- You cannot tolerate periodic breaking upgrades tied to a pre-1.0 dependency.

## Alternatives

- tauri-apps/tao — a winit fork by the Tauri team adding native menus, system tray, and desktop-app niceties; use it when you're building a desktop app shell that needs menus winit doesn't provide.
- libsdl-org/SDL (via Rust-SDL2/rust-sdl2 bindings) — batteries-included windowing plus audio, gamepad, and 2D rendering; use it when you want one mature C library covering everything, including console-style targets.
- glfw/glfw (via PistonDevelopers/glfw-rs) — a small C library for desktop OpenGL/Vulkan windowing; use it when you only target desktop and want something simpler than winit with no mobile/web.
- rust-windowing/glutin — not a replacement but the OpenGL-context companion built to sit on winit; use it when your renderer is GL rather than wgpu.
- bevyengine/bevy — if you actually want a full engine rather than a window; Bevy uses winit internally, so you rarely pick between them.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016 | Initial release; repo created Feb 2016[^3]. |
| 0.20 | 2020 | Event-loop redesign — `ControlFlow`, poll/wait model. |
| 0.28 | 2023 | `raw-window-handle` 0.5 interop, DPI split into `dpi` crate. |
| 0.29 | 2023 | Keyboard overhaul — `KeyEvent`, physical vs logical keys. |
| 0.30 | 2024 | `ApplicationHandler` trait + `run_app`; closure loop deprecated[^2]. |
| 0.31.0-beta | 2026 | In beta; MSRV raised to 1.85[^1]. |

## References

[^1]: winit README and MSRV policy, rust-windowing/winit. https://github.com/rust-windowing/winit
[^2]: `winit::application::ApplicationHandler` and `EventLoop::run_app`, docs.rs. https://docs.rs/winit/latest/winit/application/trait.ApplicationHandler.html
[^3]: GitHub repository metadata (created 2016-02-23), rust-windowing/winit. https://github.com/rust-windowing/winit

## Tags

rust, windowing, gui, event-loop, cross-platform, wayland, x11, wasm, graphics, input-handling, raw-window-handle, game-dev
