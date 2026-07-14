# iced-rs/iced

> A cross-platform GUI library for Rust that ports The Elm Architecture to a retained widget tree, rendered on wgpu.

[GitHub repo](https://github.com/iced-rs/iced) ·
[Official website](https://iced.rs) ·
[License: MIT](https://github.com/iced-rs/iced/blob/master/LICENSE)

## Overview

Iced is a GUI toolkit for Rust, started in 2019 by Héctor Ramón (`hecrj`) as an
offshoot of the Coffee 2D game library[^1]. It structures applications around
The Elm Architecture: `State`, `Message`, a `view` function that maps state to a
widget tree, and an `update` function that folds messages back into state. There
is no callback spaghetti and no mutable widget handles held across frames — the
view is rebuilt from state on every update, and iced diffs the result against the
retained tree.

The defining tension is maturity versus ambition. Iced is one of the most-starred
pure-Rust GUI libraries (30,972 stars, 1,605 forks) and is the foundation of
System76's COSMIC desktop environment (via the `libcosmic` fork) and real desktop
apps like the Halloy IRC client[^2]. Yet the README still calls it "experimental
software," and after seven years it has not reached 1.0. Every minor release
(0.x) is allowed to break API, and the layout, styling, and application-entry
APIs have each been reworked more than once. You get a clean reactive model and a
GPU renderer; you also accept a moving target.

The second tradeoff is rendering. Iced draws its own widgets on the GPU rather
than wrapping native controls, so a button looks identical on Windows, macOS, and
Linux — and identically non-native on all three. There is no OS-drawn text field,
no platform menu bar, and — historically — weak assistive-technology support.

## Getting Started

```bash
cargo add iced
```

```rust
// A complete counter. iced::run wires update + view into a windowed app.
use iced::widget::{button, column, text};

#[derive(Default)]
struct Counter { value: i32 }

#[derive(Debug, Clone, Copy)]
enum Message { Increment, Decrement }

impl Counter {
    fn update(&mut self, message: Message) {
        match message {
            Message::Increment => self.value += 1,
            Message::Decrement => self.value -= 1,
        }
    }

    fn view(&self) -> iced::Element<'_, Message> {
        column![
            button("+").on_press(Message::Increment),
            text(self.value).size(50),
            button("-").on_press(Message::Decrement),
        ]
        .into()
    }
}

fn main() -> iced::Result {
    iced::run(Counter::update, Counter::view)
}
```

For async work, side effects, or subscriptions, `update` returns a `Task`, and
`iced::application(...)` (the builder form) exposes `subscription`, `theme`, and
window configuration hooks.

## Architecture / How It Works

Iced is deliberately split into layers so the runtime can be reused outside the
default desktop shell:

- **`iced_core`** — geometry, colors, layout primitives, the `Widget` trait.
- **`iced_runtime`** — the Elm loop, `Task`/`Command`, and `Subscription` (a
  stream-of-events abstraction backed by `futures`).
- **Renderers** — `iced_wgpu` targets Vulkan, Metal, and DX12 through `wgpu`;
  `iced_tiny_skia` is a CPU software fallback for machines without a usable GPU.
  The renderer is chosen at runtime, and text is laid out with `cosmic-text`.
- **`iced_winit`** — the windowing shell, wiring `winit` events into the runtime.

Each frame, `view` produces a fresh `Element` tree. Iced runs a layout pass
(a Flexbox-like solver over rows, columns, and containers), diffs against the
previous tree to preserve widget-local state (scroll offsets, text cursors), then
emits draw primitives to the active renderer. This "rebuild the view every time"
model is cheap in Rust because `Element`s are lightweight and dropped each frame;
the cost you pay is that view functions must stay allocation-conscious in hot UIs.

Custom widgets implement the `Widget` trait directly (layout, draw, event
handling), which is more code than a callback but gives full control. Styling is
done through `Theme` and per-widget style closures rather than a stylesheet; the
styling API is one of the parts that has changed shape between releases.

## Production Notes

- **Pre-1.0, and it means it.** Upgrading a minor version (e.g. 0.12 → 0.13)
  routinely requires code changes. 0.13 replaced the old `Application`/`Sandbox`
  traits with the function-based `iced::run` / `iced::application` entry points
  shown above[^3]; projects on older tutorials will not compile. Pin the exact
  version and read the changelog before bumping.
- **GPU is the happy path.** `iced_wgpu` needs a working Vulkan/Metal/DX12 stack.
  In VMs, over remote desktop, on old drivers, or in headless CI, initialization
  can fail; the `tiny_skia` software renderer is the fallback, at a performance
  cost. Test on the environments you actually ship to.
- **Non-native everything.** No native menus, no native file dialogs (use `rfd`),
  no system context menus. Accessibility has historically been a weak spot — do
  not assume screen-reader support without verifying against your current version.
  Non-Latin text and IME are much improved via `cosmic-text` but still worth testing.
- **Binary size and build times.** Pulling in `wgpu` brings a large dependency
  tree; expect long first builds and multi-megabyte release binaries relative to
  native-toolkit bindings.
- **Ecosystem fragmentation.** `libcosmic` (System76) is a substantial fork with
  its own widgets and release cadence. Some community widgets target it, some
  target upstream iced; the two are not drop-in compatible.

## When to Use / When Not

**Use when:**
- You want a pure-Rust, single-binary desktop app with no web runtime or C++ GUI
  toolchain.
- The Elm-style unidirectional data flow fits your app and you value type-safety
  over pixel-perfect native look.
- You need consistent rendering across OSes and are comfortable owning the styling.

**Avoid when:**
- You need native platform look-and-feel, native menus/dialogs, or strong
  out-of-the-box accessibility.
- You cannot tolerate breaking changes on minor upgrades over the app's lifetime.
- Your target lacks a reliable GPU and software rendering performance matters.
- You want a quick immediate-mode debug/tooling overlay — egui is a better fit.

## Alternatives

- emilk/egui — immediate-mode; reach for it for debug UIs, tooling, and game
  overlays where you want the simplest possible loop over polish.
- slint-ui/slint — declarative `.slint` markup with a designer; use it for
  embedded/commercial products that want a markup-driven UI and native feel.
- tauri-apps/tauri — Rust backend with a web (HTML/CSS/JS) frontend; use it when
  your team already builds web UIs and wants system webview rendering.
- DioxusLabs/dioxus — React-like RSX in Rust across web/desktop/mobile; use it if
  you prefer a component/hooks model over The Elm Architecture.
- gtk-rs/gtk4-rs — bindings to native GTK 4; use it when Linux-native look,
  accessibility, and the GNOME ecosystem matter more than cross-platform sameness.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0-alpha | 2019-09 | First public renderer-agnostic release, ggez-backed[^1]. |
| 0.1.0 | 2020 | First stable-numbered release with wgpu renderer. |
| 0.12.0 | 2023-12 | Broad widget/styling refresh; pre-functional-API era. |
| 0.13.0 | 2024-09 | Function-based API (`run`/`application`), Task model[^3]. |

Iced has remained pre-1.0 throughout; there is no 1.0 release as of mid-2026,
and the repository was last pushed on 2026-07-09, so it is actively maintained.

## References

[^1]: iced README, "Implementation details" — origin in the Coffee game library, May 2019. https://github.com/iced-rs/iced#implementation-details
[^2]: System76 COSMIC is built on a fork of iced (`libcosmic`). https://github.com/pop-os/libcosmic
[^3]: iced changelog / release notes for the function-based API introduced in 0.13. https://github.com/iced-rs/iced/blob/master/CHANGELOG.md

## Tags

rust, gui, desktop-gui, cross-platform, elm-architecture, wgpu, retained-mode, widget-toolkit, reactive, native-ui
