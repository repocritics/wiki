# slint-ui/slint

> A declarative GUI toolkit that compiles a `.slint` markup language to native code, with bindings for Rust, C++, JavaScript, and Python.

[GitHub repo](https://github.com/slint-ui/slint) ·
[Official website](https://slint.dev) ·
[License: multi (Royalty-free / GPL-3.0-only / Commercial)](https://github.com/slint-ui/slint/blob/master/LICENSE.md)

## Overview

Slint is a retained-mode GUI toolkit built around its own declarative markup language, `.slint`, which describes the element tree, property bindings, layouts, and state transitions of a UI. The markup is compiled ahead of time — into Rust source, a C++ header, or interpreted at runtime for the dynamic-language bindings — and the business logic is written separately in Rust (the primary and most complete binding), C++, JavaScript/Node (beta), or Python (beta). It is developed by SixtyFPS GmbH, a small remote company in Germany, Finland, and the US; the project started in 2020 under the name SixtyFPS and was renamed to Slint in 2021[^1].

The defining goal is running the same UI stack from microcontrollers to desktops. Slint ships a CPU-only software renderer with no external dependencies (suitable for MCUs with no GPU and little RAM) alongside GPU-accelerated renderers, and it lays a component's items and properties into a single contiguous memory region to minimize allocation. This embedded-first design is the main thing that distinguishes it from Qt, Flutter, or web-based toolkits, and it drives most of its tradeoffs.

The other defining tension is licensing. Slint is open source but commercially governed: it is offered under a choice of a royalty-free license (free proprietary desktop/mobile/web apps), GPLv3 (open-source use), or a paid commercial license — the last effectively required for proprietary *embedded* products[^2]. This open-core model funds development but means the "free" story is conditional on your deployment target, which is the first thing to evaluate before adopting it.

## Getting Started

```bash
cargo add slint
```

```rust
// main.rs — UI inlined via the slint! macro (Rust binding)
slint::slint! {
    import { Button, VerticalBox } from "std-widgets.slint";
    export component MainWindow inherits Window {
        callback clicked;
        VerticalBox {
            Button {
                text: "Click me";
                clicked => { root.clicked(); }
            }
        }
    }
}

fn main() {
    let ui = MainWindow::new().unwrap();
    ui.on_clicked(|| println!("clicked!"));
    ui.run().unwrap();
}
```

Larger projects put UI in separate `.slint` files and compile them with a `build.rs` step (`slint-build`) rather than the inline macro. For non-Rust languages, the same `.slint` files are consumed via the C++ CMake integration, the `slint-ui` npm package, or the `slint` PyPI package.

## Architecture / How It Works

The pipeline is a real compiler. The `.slint` compiler runs lexing, parsing, and optimization phases, then hands off to a language-specific backend: the Rust generator emits Rust, the C++ generator emits a header, and a bundled interpreter serves the dynamic bindings. Because bindings in `.slint` are pure expressions, the compiler can inline properties and drop ones that are constant or never change, which is what keeps generated code small enough for embedded targets.

At runtime, a property engine tracks dependencies between bindings and re-evaluates them lazily when inputs change. A component's elements, items, and properties are packed into one memory region to reduce allocations — important on constrained hardware but also part of why the object model is less dynamic than a typical scene graph.

Rendering is pluggable and selected at compile time:

- **`software`** — CPU rasterizer, no GPU or external dependencies; the path for microcontrollers.
- **`femtovg`** — OpenGL ES 2.0.
- **`skia`** — Google's Skia; the heaviest but most capable renderer.

When Qt is present on the system a `qt` style becomes available, delegating widget drawing to Qt's `QStyle` for native-looking desktop widgets. Widget styling is otherwise provided by a bundled `std-widgets` set with Fluent/Material/Cupertino-flavored themes rather than by truly native OS controls, so "native" here means native *code and integration*, not necessarily pixel-identical OS widgets.

Tooling is a genuine strength: an LSP server powers autocomplete and a live preview across editors, bundled into a VS Code extension; `slint-viewer` renders `.slint` files with `--auto-reload`; SlintPad is a browser playground; and a Figma-to-Slint plugin exports designs. The live preview is the day-to-day selling point.

## Production Notes

**Licensing is the first gate, not an afterthought.** Desktop/mobile/web proprietary apps can ship free under the royalty-free license, but proprietary embedded deployment generally requires a paid commercial license, and GPLv3 is the only other option for embedded[^2]. Vet your exact target and distribution model against the current license text before committing; this is the most common source of surprise.

**Language binding maturity is uneven.** Rust is first-class and drives the design; C++ is well-supported. JavaScript/Node and Python are labeled beta and lag in API surface, documentation, and stability. Treat the beta bindings as usable-but-moving, not as parity with Rust.

**Embedded is the sweet spot and the sharp edge.** The software renderer runs on MCUs, but you own the platform integration: display drivers, an event loop, and a `PlatformWindowAdapter`/`slint::platform` implementation for bare-metal or RTOS targets. On desktop this is handled for you; on embedded it is real work, and RAM/flash budgeting for fonts and image assets matters.

**Renderer choice has real cost.** Skia pulls in a large dependency and inflates binary size; femtovg needs a working GL ES 2.0 context; the software renderer is portable but CPU-bound and slower for complex animated scenes. There is no single default that is right for both a Cortex-M board and a Skia-backed desktop app.

**The `.slint` language is its own thing.** It is not JSX, QML, or CSS, though it borrows ideas from each. Teams pay a learning curve for the property/binding/state model, and dynamic UI that does not fit the declarative component model (heavily data-driven lists, arbitrary runtime tree construction) can be awkward compared to imperative toolkits.

**API stability is a stated commitment.** Slint follows a stable 1.x API and the maintainers advertise careful evolution without breaking changes[^3] — a meaningful contrast with faster-moving UI ecosystems, and one of the better reasons to pick it for long-lived products.

## When to Use / When Not

**Use when:**
- You are targeting embedded/MCU displays and want one UI stack that also runs on desktop and web.
- Rust or C++ is your application language and you want compiled, allocation-light UI.
- You value a live-preview design loop and a declarative language over hand-written widget code.
- You need API stability over a multi-year product lifetime.

**Avoid when:**
- You are shipping proprietary embedded firmware and cannot budget a commercial license.
- Your primary language is JavaScript or Python and you need production-grade, fully stable bindings today.
- You want pixel-perfect native OS widgets rather than themed toolkit widgets.
- Your UI is highly dynamic/data-driven in ways that resist a declarative component model.

## Alternatives

- flutter/flutter — use instead when you want a mature cross-platform toolkit with the largest widget/ecosystem and can accept a heavier runtime and Dart.
- emilk/egui — use instead for immediate-mode Rust GUI in tools/debug overlays where retained-mode declarative structure is unnecessary.
- iced-rs/iced — use instead when you want a pure-Rust Elm-architecture GUI without a separate markup language or commercial licensing.
- qt/qtbase — use instead when you need the broadest desktop/embedded native widget set and QML tooling and can navigate Qt's own licensing.
- microsoft/react-native — use instead when your team is web/JS-first and wants to reuse the React model on mobile.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.x | 2020–2021 | Initial releases under the SixtyFPS name[^1]. |
| rename | 2021 | Project renamed from SixtyFPS to Slint[^1]. |
| 0.2–0.3 | 2022 | Language and binding maturation; C++/Rust bindings; embedded renderer work. |
| 1.0 | 2023-04 | First stable release; commitment to a stable 1.x API[^3]. |

## References

[^1]: SixtyFPS GmbH / Slint project — "About us" and rename history, project README and website. https://slint.dev
[^2]: Slint licensing options and FAQ — royalty-free, GPLv3, and commercial license choices; commercial license for proprietary embedded. https://slint.dev/pricing
[^3]: Slint README — stable 1.x API commitment ("We evolve carefully without breaking your code"). https://github.com/slint-ui/slint

## Tags

rust, cpp, gui-toolkit, declarative-ui, embedded, cross-platform, native-ui, wasm, lsp-server, ui-framework
