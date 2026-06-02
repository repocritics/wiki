# mrdoob/three.js

The JavaScript 3D library — the dominant WebGL / WebGPU / WebXR abstraction, used by everything from product configurators to game engines to data-visualization dashboards.

## What it is

A 3D graphics library that abstracts WebGL (and increasingly WebGPU) into a scene-graph API: cameras, lights, meshes, materials, geometries, animations, and post-processing all composed in idiomatic JavaScript. Created by Mr.doob (Ricardo Cabello) in 2010; has been the default "I want 3D in the browser" library essentially since browsers gained WebGL. Powers a long list of well-known interactive sites and tools.

## Key features

- WebGL, WebGPU, and WebXR (AR/VR) backends.
- Scene-graph API with cameras, lights, meshes, materials, geometries, animations.
- Built-in loaders for GLTF, OBJ, FBX, USDZ, and many other 3D formats.
- Post-processing pipeline (bloom, SSAO, depth-of-field, etc.).
- Physics integration via third-party libraries (rapier, ammo).
- Editor/examples site at threejs.org with hundreds of runnable demos.
- MIT-licensed.

## Tech stack

- JavaScript primary.
- TypeScript types bundled.
- npm package: `three`.
- Default branch is `dev` — releases come from tags on stabilized commits.

## When to reach for it

- You want 3D content rendered in a browser.
- You're building product configurators, data viz, or interactive web experiences with 3D elements.
- You're a game developer prototyping in the browser or shipping web-distributable games.
- You want a starting point for WebXR experimentation.

## When *not* to reach for it

- You want declarative 3D — React Three Fiber wraps Three with React idioms.
- You want a full game engine — Babylon.js has more game-engine-shaped features built in; PlayCanvas is editor-first.
- You want native (non-browser) 3D — Three.js is browser-targeted.

## Maturity signal

113k stars, 36k forks, MIT, last push the day this page was generated. 14-year-old project — among the oldest JS libraries still in active development. Open-issues count of 435 is moderate. The `dev` default branch + tagged releases is the project's stability pattern: continuous integration on dev, semver-tagged releases for users.

## Alternatives

- React Three Fiber (`pmndrs/react-three-fiber`) — use when you want React-component-flavored Three.js.
- BabylonJS — use when you want a more game-engine-shaped WebGL framework.
- PlayCanvas — use when you want a hosted, editor-first 3D engine.
- Bevy (Rust + WebAssembly) — use when you want code-first ECS 3D in the browser via WASM.

## Notes

The "Three.js is the safe default for browser 3D" position is durable — 14 years of compatibility, broad ecosystem, MIT license, and a healthy plugin landscape. WebGPU support is the most-significant recent architectural change; current code paths support both WebGL and WebGPU backends.

## Tags

javascript, typescript, 3d, graphics, webgl, webgpu, webxr, library, frontend, three-js
