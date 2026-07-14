# BabylonJS/Babylon.js

> A full-stack 3D engine for the web — renderer, scene graph, materials, physics, GUI, and asset loaders in one TypeScript framework.

[GitHub repo](https://github.com/BabylonJS/Babylon.js) ·
[Official website](http://www.babylonjs.com) ·
[License: Apache-2.0](https://github.com/BabylonJS/Babylon.js/blob/master/license.md)

## Overview

Babylon.js is a WebGL/WebGPU 3D engine started inside Microsoft by David Catuhe in 2013 and open-sourced under Apache-2.0[^1]. Unlike a bare renderer, it ships a complete runtime: scene graph, cameras, PBR and node materials, animation, a full GUI system, physics plugins, glTF loading, an in-browser inspector, and WebXR support — all with first-class TypeScript typings. It targets developers building 3D product configurators, data viz, simulations, browser games, and immersive (VR/AR) experiences without assembling a stack of separate libraries.

The defining tension is **batteries-included vs. weight**. Babylon's completeness is its selling point — most of what a 3D app needs is in the box and documented — but the UMD bundle (`babylonjs`) is large, and the engine assumes you will adopt its abstractions (its `Scene`, its material system, its animation groups) rather than compose your own. The counterweight is the ES6 package family (`@babylonjs/core` and siblings), which is tree-shakeable and is the only sane choice for production bundle size[^2].

Governance sits with Microsoft but development is genuinely community-driven: a large contributor base, an active Discourse forum, and a Playground (playground.babylonjs.com) where every doc example is a runnable, shareable snippet. The project is unusual among Microsoft-originated OSS in being permissively licensed and led primarily through the open repo rather than a product team.

## Getting Started

```bash
# ES6, tree-shakeable — recommended for production
npm install @babylonjs/core
# or the UMD single-bundle (larger, convenient for prototypes)
npm install babylonjs
```

```ts
import { Engine, Scene, ArcRotateCamera, HemisphericLight, Vector3, MeshBuilder } from "@babylonjs/core";

const canvas = document.getElementById("renderCanvas") as HTMLCanvasElement;
const engine = new Engine(canvas, true);
const scene = new Scene(engine);

const camera = new ArcRotateCamera("cam", -Math.PI / 2, Math.PI / 2.5, 6, Vector3.Zero(), scene);
camera.attachControl(canvas, true);
new HemisphericLight("light", new Vector3(0, 1, 0), scene);
MeshBuilder.CreateSphere("sphere", { diameter: 2 }, scene);

engine.runRenderLoop(() => scene.render());
window.addEventListener("resize", () => engine.resize());
```

For a zero-install first look, the Playground pre-wires the engine and scene so you only write scene code.

## Architecture / How It Works

The runtime is a two-object core. An **`Engine`** owns the graphics context (WebGL, WebGL2, or WebGPU) and the render loop; a **`Scene`** owns the scene graph — meshes, cameras, lights, materials, and the animation/observable system. You typically hold one `Engine` and one or more `Scene`s and call `scene.render()` each frame inside `engine.runRenderLoop`.

Notable internals:

- **Renderer abstraction.** `Engine` (WebGL) and `WebGPUEngine` share an interface, so most application code is API-agnostic. WebGPU support went GA in 5.0[^3] but WebGL2 remains the safe default; not every feature path is identical across the two backends.
- **Materials.** `StandardMaterial` (Blinn-Phong-era), `PBRMaterial` (physically based), and `NodeMaterial` — a graph-based shader system with a visual editor that compiles to GLSL/WGSL. Shaders are generated and cached per-material-configuration.
- **Meshes and geometry.** Thin instances, hardware instancing, and mesh merging are the primary draw-call reduction tools. `MeshBuilder` is the factory for primitives.
- **Physics.** Plugin-based. The current path is the **Havok** engine compiled to WebAssembly, exposed through the v2 physics API introduced in 6.0[^4]; older Cannon.js and Ammo.js plugins use the v1 API.
- **Loaders and GUI.** glTF/GLB is the primary asset format (via `@babylonjs/loaders`); the GUI (`@babylonjs/gui`) is a retained-mode 2D/3D UI layer rendered into the scene, not DOM.

The repo is a monorepo (`packages/dev/*`) where `core` is the engine and `gui`, `materials`, `loaders`, `serializers`, `inspector`, and the node/GUI editors are separate publishable packages. This is why two parallel npm worlds exist: the legacy UMD `babylonjs*` packages and the modern scoped `@babylonjs/*` ES6 packages.

## Production Notes

**Pick one package world and never mix.** Importing from both `babylonjs` (UMD) and `@babylonjs/core` (ES6) in the same app duplicates the engine and produces subtle "instanceof" and singleton bugs. Standardize on `@babylonjs/*` for anything you ship.

**Bundle size is the recurring surprise.** The UMD `babylonjs` bundle is multi-megabyte because it contains everything. Tree-shaking only works with the ES6 packages *and* side-effect-free imports; pulling in `MeshBuilder` transitively drags large chunks unless you import specific builders and enable your bundler's tree-shaking. Budget for this early rather than discovering it at launch.

**The CDN is not for production.** The project's own README warns that cdn.babylonjs.com exists for learning and experiments; self-host your pinned version[^5].

**WebGPU is real but not free.** It unlocks compute-driven features and better draw-call handling, but browser coverage, feature parity with the WebGL2 path, and shader-compilation timing differ. Treat WebGPU as opt-in and keep the WebGL2 fallback tested.

**Upgrade friction lives at the physics boundary.** The v1→v2 physics API (Havok) is the most disruptive migration in recent memory; a project on Cannon/Ammo v1 does not transparently move to Havok v2. Major versions have shipped roughly yearly and are generally additive, but the physics and, over time, material/shader defaults are where breaks concentrate.

**The Inspector is a debug tool, not a shipping dependency.** `@babylonjs/inspector` is invaluable for scene debugging but heavy; load it lazily/conditionally, never in the production bundle.

## When to Use / When Not

**Use when:**
- You want a complete engine (materials, physics, GUI, glTF, XR) without integrating five libraries yourself.
- You need WebXR / VR / AR on the web with a maintained, documented path.
- Your team values TypeScript typings, an interactive Playground, and thorough docs over minimal footprint.
- You're building configurators, simulations, or browser games where an editor-grade material/node system pays off.

**Avoid when:**
- Bundle size is your hard constraint and you only need to render a few models — a lower-level renderer will ship smaller.
- You want a full desktop/mobile/console game engine with a native editor and export targets beyond the web.
- You prefer a declarative React-first authoring model as the primary API.
- You need the largest possible pool of examples/tutorials for a specific niche — the general-purpose 3D-web community skews larger elsewhere.

## Alternatives

- mrdoob/three.js — use instead when you want a lower-level renderer and the largest community, and are willing to add materials, physics, and loaders yourself.
- playcanvas/engine — use instead when you want a cloud-hosted visual editor and a game-first workflow out of the box.
- pmndrs/react-three-fiber — use instead when your app is React and you want declarative 3D (renders three.js under the hood).
- godotengine/godot — use instead when you need a full game engine with a native editor and non-web export targets.
- aframevr/aframe — use instead when you want declarative HTML-based WebXR scenes rather than an imperative engine API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Open-sourced from Microsoft under Apache-2.0[^1]. |
| 2.0 | 2015 | Physics, LOD, procedural textures, expanded material system. |
| 3.0 | 2017 | WebVR, morph targets, PBR maturity. |
| 4.0 | 2019-05 | Node Material Editor, improved PBR, inspector. |
| 5.0 | 2022-04 | WebGPU support (GA), snapshot rendering, performance work[^3]. |
| 6.0 | 2023-04 | Havok physics + v2 physics API, performance priority modes[^4]. |
| 7.0 | 2024 | IBL shadows, Gaussian splatting, further WebGPU work. |
| 8.0 | 2025 | Continued WebGPU/rendering pipeline improvements. |

## References

[^1]: Babylon.js origin and Apache-2.0 licensing; repo created 2013-06-27. https://github.com/BabylonJS/Babylon.js
[^2]: Babylon.js docs, "ES6 / npm support and tree shaking". https://doc.babylonjs.com/setup/frameworkPackages/npmSupport
[^3]: Babylon.js docs, "WebGPU support". https://doc.babylonjs.com/setup/support/webGPU
[^4]: Babylon.js docs, "Physics engine (v2) and Havok". https://doc.babylonjs.com/features/featuresDeepDive/physics
[^5]: Babylon.js README CDN warning — "The CDN should not be used in production environments." https://github.com/BabylonJS/Babylon.js#cdn

## Tags

typescript, javascript, 3d, game-engine, rendering-engine, webgl, webgpu, webxr, gltf, physics, web-graphics, pbr
