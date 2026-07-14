# emilk/egui

> An immediate-mode GUI library in pure Rust that runs the same code on native desktop and the web.

[GitHub repo](https://github.com/emilk/egui) ·
[Official website](https://www.egui.rs/) ·
[License: MIT OR Apache-2.0](https://github.com/emilk/egui/blob/main/LICENSE-APACHE)

## Overview

egui (pronounced "e-gooey") is an immediate-mode GUI library written by Emil Ernerfeldt, first published to crates.io in 2020 after starting as a personal project named "emigui" in 2019[^1]. It is the most-adopted pure-Rust GUI toolkit as of 2026 — ~29.6k stars, ~2.1k forks, and a steady multi-year release cadence[^2]. Its central bet is immediate mode: you do not build a widget tree and register callbacks; you call `ui.button("Save")` every frame and read its return value inline. This collapses most GUI state-synchronization bugs (stale views, dangling callbacks, out-of-sync app/UI state) because the UI stores almost nothing between frames.

egui deliberately knows nothing about windows, input, or the GPU. It consumes raw input and produces a list of textured triangles plus an output struct; an *integration* (backend) is responsible for feeding input and painting the mesh. The 2D tessellation and font layer is a separate crate, `epaint`. The official all-in-one framework is `eframe`, which wires egui to `winit` and a renderer (`egui_glow` or `egui-wgpu`) so one codebase compiles to Windows/macOS/Linux/Android and to WebAssembly.

The defining tension is stated bluntly in the project's own README: egui optimizes for ease of use over power, and its interfaces are still in flux with breaking changes in most releases. It has not reached 1.0. It does not try to look native, and it re-runs layout every frame — trades that are cheap for tool UIs and dashboards but constraining for large, text-heavy, or accessibility-critical applications.

## Getting Started

```bash
cargo new my-app
cd my-app
cargo add eframe
```

```rust
// src/main.rs — a full eframe app; the closure runs every frame.
use eframe::egui;

fn main() -> eframe::Result {
    let opts = eframe::NativeOptions::default();
    let mut name = "world".to_owned();
    let mut age = 30u32;

    eframe::run_simple_native("My egui App", opts, move |ctx, _frame| {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading("My egui Application");
            ui.horizontal(|ui| {
                ui.label("Your name:");
                ui.text_edit_singleline(&mut name);
            });
            ui.add(egui::Slider::new(&mut age, 0..=120).text("age"));
            if ui.button("Increment").clicked() {
                age += 1;
            }
            ui.label(format!("Hello '{name}', age {age}"));
        });
    })
}
```

For the web, use the `eframe_template` repository, which builds to Wasm and serves via Trunk. On Linux, the wgpu/glow demo needs system packages (`libgtk-3-dev`, `libxkbcommon-dev`, xcb libs, etc.) — egui itself is platform-agnostic, but the backend is not.

## Architecture / How It Works

The frame loop is the whole model. Each frame the integration calls `Context::run` (or `begin_pass`/`end_pass`) with a `RawInput`; your closure issues widget calls; egui returns `FullOutput` containing a tessellated `ClippedPrimitive` mesh, texture deltas, and platform commands (cursor, clipboard, open-URL). The integration paints the mesh however it can draw textured triangles.

Crate layout, roughly bottom-up:

- **`epaint`** — fonts (via `ab_glyph`), text layout, anti-aliased tessellation of shapes into vertex/index buffers. No windowing.
- **`egui`** — the widget and layout library: `Context`, `Ui`, panels, windows, the `Id` system, styling. Contains no `unsafe` (`#![forbid(unsafe_code)]`).
- **`egui-winit`** — translates `winit` events into egui `RawInput`.
- **`egui_glow`** (OpenGL/WebGL via `glow`) and **`egui-wgpu`** (wgpu/WebGPU) — the two official painters. wgpu is the more actively developed path.
- **`eframe`** — bundles the above into a single entry point plus `image`, clipboard, and other heavier deps.

**The `Id` system** is how a stateless-per-frame library keeps the few things that must persist: window positions, scroll offsets, which widget is being dragged. Widget IDs are auto-derived from call-site and label; retained-state containers (windows, collapsing headers) seed IDs from their title. Two windows with the same title collide and need an explicit `Id` source — the rare place immediate mode leaks into user code.

**The immediate-mode layout paradox** is the honest structural weakness. To position a centered window egui must know its size, but sizing requires laying out the contents, and layout also handles interaction — so position must be decided before size is known. egui resolves this by caching last-frame size and reusing it, which causes one-frame jitter when content first appears. For cases that must be pixel-correct on the first frame, `Context::request_discard` runs a second layout pass and discards the first; egui avoids this by default because of the doubled CPU cost, so most frames stay single-pass[^3].

## Production Notes

- **It re-lays-out every frame.** CPU cost is typically 1–2 ms/frame for normal UIs, but a very large widget count inside a scroll area is O(content) per frame. The fix is virtualization: `ScrollArea::show_rows` / `show_viewport` lay out only visible rows. Ignoring this is the most common performance complaint.
- **It only repaints on demand.** egui does not spin at 60 Hz when idle; it repaints on input, animation, or an explicit `ctx.request_repaint()`. Background work (a finished network request on another thread) will not appear until something requests a repaint — a frequent "my UI is frozen until I move the mouse" bug.
- **No async in UI code.** Awaiting in the frame closure blocks the UI thread. The documented pattern is to run work on another thread/task and communicate via channels (`try_recv`, never blocking), `Arc<Mutex<_>>`, or `poll_promise`.
- **Breaking changes are routine.** Pre-1.0 releases regularly change APIs and the visual style; pinning an exact `egui`/`eframe` version and reading the CHANGELOG before bumping is standard practice. All egui-ecosystem crates must share the same egui version — mismatched `bevy_egui`/`egui_extras`/`egui` versions fail to compile against the re-exported types.
- **Accessibility is partial.** AccessKit integration exposes native a11y trees on Windows and macOS (enabled by default in eframe); Linux and web are weaker, with only an experimental built-in screen reader on web. Do not assume screen-reader parity across platforms.
- **Text and fonts.** Non-Latin scripts require shipping your own font via `Context::set_fonts`; there is no system-font fallback by default, and complex-script shaping is limited compared to a native toolkit.
- **Styling is not CSS.** Colors, spacing, and sizes are set through `Style`/`Visuals`, which is flexible for theming but not a full layout/skinning system.

## When to Use / When Not

**Use when:**
- You are writing in Rust and want a GUI with minimal ceremony.
- You need one codebase to run natively and in the browser (Wasm).
- You are adding tooling/debug/editor UI on top of a game engine or renderer that already draws triangles.
- Your UI is interactive and data-driven (dashboards, viewers, dev tools), where immediate mode's simplicity pays off.

**Avoid when:**
- You need a native look and platform-standard widgets — egui draws its own.
- You need mature, cross-platform accessibility guarantees.
- You want long-term API stability with rare breaking changes — egui is still 0.x.
- You are rendering huge static documents or very large scrollbacks where per-frame layout is wasteful and a retained-mode toolkit fits better.

## Alternatives

- iced-rs/iced — retained/Elm-style Rust GUI with a message-and-update architecture; pick it when you want a reactive model and more native-feeling layout instead of per-frame immediate mode.
- slint-ui/slint — declarative `.slint` markup with a native-leaning renderer and commercial support; use when you want designer-friendly UI files and embedded targets.
- ocornut/imgui (with imgui-rs bindings) — the C++ immediate-mode toolkit egui is philosophically modeled on; use when you need its vast widget/plugin ecosystem or C/C++ interop.
- xilem (linebender) — experimental Rust reactive UI from the Druid lineage; consider when you want a research-forward retained architecture and can tolerate churn.
- tauri-apps/tauri — ship the UI as web tech in a system webview with a Rust backend; choose when your frontend is HTML/CSS/JS and you only want Rust for the native layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-01 | Repo created as "emigui", Emil Ernerfeldt's personal project[^1]. |
| 0.1.0 | 2020 | Renamed egui, first crates.io release. |
| 0.20 | 2022-12 | AccessKit accessibility integration merged[^4]. |
| 0.24 | 2023-12 | Multiple native viewports (multi-window) support[^4]. |
| 0.x | ongoing | Regular releases, breaking changes each version; no 1.0 yet[^2]. |

## References

[^1]: egui README and repository history, emilk/egui. https://github.com/emilk/egui
[^2]: GitHub repository metadata (stars, forks, last push 2026-07-08), fetched via GitHub API. https://github.com/emilk/egui
[^3]: egui docs, "Understanding immediate mode" and the layout-size discussion in issue #4378. https://docs.rs/egui/latest/egui/#understanding-immediate-mode
[^4]: egui CHANGELOG. https://github.com/emilk/egui/blob/main/CHANGELOG.md

## Tags

rust, gui, immediate-mode, wasm, webassembly, cross-platform, desktop-ui, gamedev, egui, eframe, graphics
