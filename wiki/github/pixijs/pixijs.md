# pixijs/pixijs

PixiJS — the fastest, most flexible 2D WebGL renderer. The HTML5 game / data-visualization workhorse.

## What it is

A TypeScript library that wraps WebGL into a high-performance 2D rendering API: sprites, text, graphics, filters, particle effects, scene graph, interactive picking. Used heavily for browser games, interactive data viz, and ad creative. Doesn't include game-engine features (physics, audio, asset pipeline) — those plug in via sister libraries.

## Key features

- WebGL + WebGPU (newer versions) rendering with WebGL fallback to Canvas.
- Sprite-based scene graph with transforms, interactivity.
- Built-in filters (blur, color matrix, displacement, etc.) and custom GLSL shaders.
- Text rendering (HTML-canvas-backed + bitmap fonts).
- Asset loader with progress events.
- TypeScript types bundled.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- WebGL / WebGPU rendering.

## When to reach for it

- You're building browser games or interactive 2D visualizations needing performance.
- You want a renderer-only library (compose your own engine on top).

## When *not* to reach for it

- You need a full game engine — Phaser, Babylon, Godot HTML5 export are closer-fit.
- You need 3D — Three.js or Babylon.js for that.

## Maturity signal

47k stars, 5k forks, MIT, actively maintained. 12+ years.

## Alternatives

- Phaser — game framework on top of PixiJS-style rendering.
- Three.js — for 3D.
- HTML5 Canvas / SVG directly — for simple cases.

## Tags

typescript, javascript, webgl, rendering, 2d, library, game, mit-license, frontend
