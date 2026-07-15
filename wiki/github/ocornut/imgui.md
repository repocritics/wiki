# ocornut/imgui

> Immediate-mode GUI for C++ that emits vertex buffers instead of owning a window — the de facto standard for game-engine tooling and debug UI.

[GitHub repo](https://github.com/ocornut/imgui) ·
[Website](https://www.dearimgui.com) ·
[License: MIT](https://github.com/ocornut/imgui/blob/master/LICENSE.txt)

## Overview

Dear ImGui is a C++ GUI library built on the immediate-mode paradigm: instead of constructing a retained widget tree that you mutate over time, you re-declare the entire UI every frame with plain function calls, and the library figures out what changed[^1]. It was written by Omar Cornut, open-sourced in 2014, and has become the default choice for in-engine tools, debug overlays, and content-creation UIs across the game industry. It is explicitly *not* aimed at end-user application UIs — there is no internationalization (no right-to-left or bidirectional text, no text shaping) and no accessibility layer[^2].

The defining design decision is that the library touches no operating-system or GPU state. It consumes input you feed it and produces optimized vertex buffers plus a small list of draw-call batches; you render those however you like[^3]. That is what makes it renderer- and platform-agnostic — anywhere you can draw textured triangles, you can draw ImGui. The recurring misconception is that "immediate mode" means immediate-mode *rendering* (a flood of draw calls); it does not. The tradeoff you actually accept is different: because UI state is reconstructed each frame and keyed by an ID stack rather than by object identity, subtle bugs come from ID collisions and layout that depends on frame-to-frame ordering, not from retained-tree desync.

The project is single-maintainer-led and deliberately conservative about scope. It favors being easy to integrate, easy to hack, and cheap at runtime over breadth of widgets. It is battle-tested but permanently under-resourced relative to its reach, which is why the README openly solicits corporate funding.

## Getting Started

There is no build system to adopt — you drop the core `imgui*.cpp`/`imgui*.h` files from the repo root into your project, then add one platform backend and one renderer backend from `backends/`[^3].

```cpp
// Per-frame loop (backend init/shutdown omitted).
ImGui_ImplXXXX_NewFrame();   // e.g. imgui_impl_glfw
ImGui_ImplXXXX_NewFrame();   // e.g. imgui_impl_opengl3
ImGui::NewFrame();

ImGui::Begin("My First Tool", &open, ImGuiWindowFlags_MenuBar);
if (ImGui::Button("Save"))
    MySaveFunction();
ImGui::SliderFloat("float", &f, 0.0f, 1.0f);
ImGui::ColorEdit4("Color", my_color);   // my_color = float[4]
ImGui::End();

ImGui::Render();
ImGui_ImplXXXX_RenderDrawData(ImGui::GetDrawData());
```

Call `ImGui::ShowDemoWindow()` to open the live demo; its source in `imgui_demo.cpp` doubles as the de facto API reference. The official recommendation is to track `master` (or `docking`) rather than tagged releases — the library is stable and regressions are fixed quickly[^4].

## Architecture / How It Works

Everything lives behind a global `ImGuiContext`. Each widget derives an integer ID by hashing its label against a stack of parent IDs; that ID is how the library correlates this frame's declaration with last frame's state (open/closed, scroll position, active-edit buffer). This is the single most important mechanic to understand — two widgets with the same label under the same parent collide, and `PushID`/`##hidden`/`###fixed` label suffixes exist to disambiguate.

State that must persist between frames (window positions, tree-node open flags, table column widths) is stored in per-context storage keyed by ID, and optionally serialized to an INI file. Your application state is *not* stored by ImGui; you pass pointers in and out each frame, which is the whole point of "minimize state synchronization."

Rendering is a two-stage split. ImGui builds `ImDrawList`s (indexed triangle lists with clip rects and texture bindings) into `ImDrawData`; a backend's render function walks that and issues the actual API calls. The repo ships ~20 backends — renderers for DirectX 9–12, OpenGL/ES, Vulkan, Metal, WebGPU, SDL_GPU/SDL_Renderer, and platform layers for GLFW, SDL2/3, Win32, OSX, Android, Emscripten[^3]. Text is rasterized to an atlas texture via bundled stb_truetype; there is no external font/shaping dependency.

Two long-lived branches matter. `master` is the mainline. `docking` carries the Docking and Multi-Viewport features (windows that can be dragged out into real OS windows), kept in sync with master but held separate because those features impose extra requirements on backends and have not been promoted to the default branch[^4]. Choosing `docking` is a real architectural commitment, not a flag flip.

## Production Notes

- **Backend upgrades are the recurring tax.** The core API is stable, but the backend contract is not frozen. The 1.92 line reworked fonts into a dynamic, texture-driven system where the backend must handle texture creation/updates each frame; custom or vendored backends written against older versions need changes to compile and render correctly[^5]. If you fork a backend (most engines do), budget for periodic reconciliation.
- **ID collisions are the classic footgun.** Widgets in a loop that share a label silently share state — buttons that all fire together, input fields that stomp each other. The fix (`PushID(i)` per iteration) is trivial once you know; the symptom is baffling until you do.
- **`docking` vs `master` is a fork decision.** Adopting Multi-Viewport pulls in platform-window management and constrains which backends you can use unmodified. Migrating off it later is painful, so decide early.
- **Not thread-safe and single-context by default.** All calls belong to one context and one thread per frame. Multi-context/multi-threaded setups are possible but you own the discipline.
- **No accessibility or i18n — by design.** If your UI must serve screen readers, RTL scripts, or complex text shaping, this is the wrong tool and no amount of extension fixes it[^2].
- **Sync-to-master means you own regression testing.** The recommended workflow (track the tip) trades tagged-release predictability for fast fixes; pin a commit and test on bump if you need reproducibility. The separate Dear ImGui Test Engine exists for exactly this and is dual-licensed (free for individuals/open source, paid for business use)[^6].
- **Widget breadth is intentionally limited.** Tables, plots, trees, and color pickers are built in, but rich data grids, node graphs, and advanced plotting come from third-party extensions (ImPlot, imgui-node-editor, etc.), each with its own maintenance risk.

## When to Use / When Not

**Use when:**
- You need debug UI, tooling, or an in-engine editor for a real-time/3D/game application.
- You already have a render loop and can feed input + draw triangles.
- You want fast iteration (add a slider mid-algorithm, delete it a minute later) with almost no boilerplate.
- You value a self-contained dependency you can vendor and hack.

**Avoid when:**
- You are shipping a consumer-facing application UI that needs accessibility, localization, or complex text.
- You have no render loop of your own and want a batteries-included app framework (use Qt).
- You need a retained scene of widgets you mutate imperatively from many places.
- You want strict semantic-versioning stability guarantees on the integration surface.

## Alternatives

- ocornut/imgui remains the reference, but for a pure-C API on top of it use cimgui/cimgui when your host language isn't C++.
- microsoft/microsoft-ui-xaml or the Qt Project (qt/qtbase) — use instead when you are building an actual end-user desktop application that needs accessibility and native look.
- emilk/egui — use instead when your stack is Rust and you want an immediate-mode GUI native to that ecosystem.
- Nelarius/imnodes / thedmd/imgui-node-editor — use alongside, not instead, when you specifically need node-graph editing.
- raysan5/raygui — use instead when you want an immediate-mode-ish UI tied to raylib's minimalist C world.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-07 | Open-sourced by Omar Cornut (originally "ImGui")[^1]. |
| — | ~2018 | `docking` branch introduced for Docking + Multi-Viewport[^4]. |
| 1.80 | 2021-01 | Tables API added, replacing the old Columns system[^7]. |
| 1.89 | 2022 | Ongoing widget/backend refinements; auto-generated C bindings via dear_bindings matured. |
| 1.92 | 2025 | Dynamic, texture-driven font system; backends must handle runtime texture updates[^5]. |
| 1.92.6 | 2026-02 | Demo binaries built from `master` at this tag[^4]. |

## References

[^1]: Dear ImGui repository and README, "The Pitch" / "How it works". https://github.com/ocornut/imgui
[^2]: README scope statement: no full internationalization (RTL, bidirectional, shaping) or accessibility. https://github.com/ocornut/imgui#the-pitch
[^3]: README "Usage" and "Getting Started & Integration"; core files in repo root, backends in `backends/`. https://github.com/ocornut/imgui/tree/master/backends
[^4]: README "Which version should I get?" — recommends tracking `master`/`docking`; docking branch carries Multi-Viewport/Docking. https://github.com/ocornut/imgui/wiki
[^5]: Dear ImGui 1.92 font-system rework (dynamic fonts, backend texture-update requirement). https://github.com/ocornut/imgui/releases
[^6]: Dear ImGui Test Engine (separate repo, dual-licensed). https://github.com/ocornut/imgui_test_engine
[^7]: Tables API introduced in 1.80. https://github.com/ocornut/imgui/releases

## Tags

cpp, gui, immediate-mode, gamedev, game-engine, tooling, graphics, rendering, cross-platform, debug-ui, native
