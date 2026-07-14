# phaserjs/phaser

> A 2D HTML5 game framework with Canvas and WebGL rendering, a scene system, and two bundled physics engines.

[GitHub repo](https://github.com/phaserjs/phaser) ·
[Official website](https://phaser.io) ·
[License: MIT](https://github.com/phaserjs/phaser/blob/master/LICENSE.md)

## Overview

Phaser is a browser-first 2D game framework written in JavaScript, started by Richard Davey (Photon Storm) with the first public release in 2013[^1]. It occupies the middle of the game-tooling stack: higher-level than a raw renderer like Pixi.js (it ships a scene manager, loader, input, cameras, tilemaps, tweens, animation, and physics), but lower-level than an editor-based engine like Godot or Unity (there is no visual scene editor — everything is code). Games ship to the web and can be wrapped for iOS, Android, Steam, and newer distribution surfaces like Discord Activities and YouTube Playables via third-party tooling[^2].

The defining tension is that Phaser is a code-only framework maintained by a small commercial team (Phaser Studio Inc) against a very large, long-tailed API surface. That surface is its strength — batteries-included, one dependency, works with any front-end stack — and its liability: the API is broad, the runtime is a single monolithic module, and major versions have historically been full renderer rewrites (v2 on Pixi, v3 with a bespoke renderer, v4 with a node-based renderer) that require migration effort rather than drop-in upgrades[^3].

As of 2026 the current line is Phaser 4 (4.2.1), a major release built on a rewritten WebGL renderer. Phaser 3 remains widely deployed and is what most existing tutorials, examples, and Stack Overflow answers target, so version drift between docs and code is a real friction point.

## Getting Started

```bash
npm install phaser
# or scaffold a full project with a template + bundler
npm create @phaserjs/game@latest
```

```js
import Phaser from "phaser";

class MainScene extends Phaser.Scene {
  preload() {
    this.load.image("sky", "assets/sky.png");
  }
  create() {
    this.add.image(400, 300, "sky");
    this.add.text(400, 300, "Hello Phaser", { fontSize: "32px" })
      .setOrigin(0.5);
  }
}

new Phaser.Game({
  type: Phaser.AUTO,          // WebGL if available, else Canvas
  width: 800,
  height: 600,
  physics: { default: "arcade" },
  scene: [MainScene],
});
```

`Phaser.AUTO` picks WebGL and silently falls back to Canvas; `Phaser.WEBGL` / `Phaser.CANVAS` force a renderer. A game is a tree of Scenes, each with `preload` / `create` / `update` lifecycle hooks.

## Architecture / How It Works

**Game and Scene.** A `Phaser.Game` owns the canvas, the main loop, and global systems (loader cache, sound, input, scale manager). Actual gameplay lives in **Scenes** — self-contained units with their own display list, camera, and update loop. Scenes can run in parallel, sleep, launch, and pass data between each other; there is no global "level" object, which is why Phaser code tends to be organized by scene rather than by entity.

**Rendering.** Phaser has two backends behind one API: a Canvas 2D renderer and a WebGL renderer. In Phaser 3 the WebGL path used a "pipeline" abstraction for batching and custom shaders. Phaser 4 replaced this with a **render-node** architecture: each node performs one rendering task, WebGL state is explicitly managed, and context loss is handled gracefully[^3]. v4 also unified the v3 FX and Mask systems into a single **Filter** system that can apply to any game object or camera, and adds GPU-batch objects (`SpriteGPULayer`, `TilemapGPULayer`) that push large sprite/tile counts into a single draw call. This is the part of the API most affected by upgrades.

**Physics.** Two engines ship in-tree and are opt-in per game: **Arcade Physics** (fast AABB / circle collision, no rotation on bodies — for platformers and arcade games) and **Matter.js** (a full 2D rigid-body engine with constraints and polygons). They are not interchangeable; a body is written against one API or the other. There is no built-in continuous-collision or tile-slope solver in Arcade beyond its tile-collision helpers.

**Game objects and the display list.** Sprites, text, containers, tilemaps, particles, and shaders are all display-list nodes. Containers nest transforms. The loader (`this.load.*`) is a queued, event-driven asset pipeline with a global cache keyed by string, so asset keys are effectively a global namespace you manage yourself.

**Build shape.** Phaser is distributed as a single module. The unminified `phaser.js` is ~8 MB, but roughly 84% of that is inline JSDoc that powers the bundled TypeScript definitions; the full minified build is ~1.29 MB (~345 KB gzipped)[^4]. Custom builds can exclude unused subsystems (e.g. an Arcade-only build drops Matter).

## Production Notes

- **Bundle size is coarse-grained.** Phaser is not aggressively tree-shakeable — you pull in the framework, not just the classes you use. The escape hatch is a custom build config that excludes whole subsystems (Matter physics, specific game objects). Budget ~345 KB gzipped for the full runtime before your own assets.
- **Version/documentation drift.** The majority of tutorials, examples, and community answers target Phaser 3. Phaser 4 keeps most of the public API but changes the renderer, tint system, FX/Masks, Shader API, and removes some classes (e.g. `Point`, `Mesh`, `BitmapMask`)[^3]. Copy-pasting v3 snippets into a v4 project is a common source of breakage. There is an official migration guide and a bundled AI-agent migration skill.
- **Physics choice is a commitment.** Picking Arcade vs Matter early matters — porting a game between them is a rewrite of all body/collision code. Arcade is cheap and predictable; Matter is heavier and can be a CPU sink with many bodies.
- **Asset-key collisions.** Because the loader cache is a flat global string namespace, large projects hit silent key collisions (loading a new texture under an existing key overwrites it). Establish a naming convention early.
- **Mobile GPU variance.** WebGL behavior differs across mobile GPUs and browsers; batch breaks and texture-unit limits cause real frame-rate cliffs. v4's smarter multi-texture batching helps, but test on target devices, not just desktop.
- **Input and scale manager.** Coordinate systems (world vs screen vs camera) and the Scale Manager's resize modes are recurring beginner footguns; pointer coordinates must be translated through the active camera.
- **No editor.** There is no first-party visual scene/level editor. Tilemaps are typically authored in Tiled and imported; entity layout is code or hand-rolled data.

## When to Use / When Not

**Use when:**
- You want a code-only 2D framework for the web with physics, tilemaps, input, and audio already integrated.
- You're targeting browsers first (including Discord Activities, YouTube Playables, Reddit, itch.io) and want one dependency, not an engine install.
- Your team is comfortable in JavaScript/TypeScript and prefers version-controllable code over a proprietary editor/scene format.
- You want an API that current LLM coding agents already know well.

**Avoid when:**
- You need 3D — Phaser is 2D only (use Three.js, Babylon.js, or a full engine).
- You want a visual editor, built-in scene/prefab tooling, or native-first export — Godot or Unity fit better.
- You only need a renderer and want to build your own architecture — Pixi.js is lower-level and lighter.
- You need the smallest possible bundle for a tiny game — a micro-framework or hand-rolled Canvas beats a ~345 KB runtime.

## Alternatives

- pixijs/pixijs — use instead when you want a fast 2D WebGL/Canvas renderer only and will build your own scene, physics, and input layers.
- excaliburjs/Excalibur — use when you want a TypeScript-first, more opinionated ECS-flavored game engine with batteries included.
- melonjs/melonJS — use for a lightweight HTML5 engine with strong Tiled integration and a smaller footprint.
- godotengine/godot — use when you want a full visual editor, scene/prefab tooling, and native + web export from one project.
- kaplayjs/kaplay — use for small, jam-scale arcade games where a tiny, playful API beats a broad framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | First public release by Richard Davey / Photon Storm[^1]. |
| 2.0 | 2014 | Rendering via Pixi.js; the long-lived v2 line. |
| CE | 2016 | Phaser CE — community-maintained fork of v2 after it was frozen. |
| 3.0 | 2018-02 | Full rewrite: bespoke WebGL/Canvas renderer, new Scene system[^5]. |
| 3.60 | 2023 | Large v3 feature release (SpineGameObject, new FX, Nine Slice). |
| 4.0 | 2026 | Render-node renderer, unified Filter system, GPU sprite/tile layers[^3]. |
| 4.2.1 | 2026-07 | Current release on the v4 line[^6]. |

## References

[^1]: Phaser — history and origin, Photon Storm / Phaser Studio. https://phaser.io/about
[^2]: Phaser README — supported distribution targets (web, Discord Activities, YouTube Playables, native via third-party tools). https://github.com/phaserjs/phaser
[^3]: Phaser 4.0 Migration Guide — renderer rewrite, unified filters, removed classes. https://github.com/phaserjs/phaser/blob/master/changelog/v4/4.0/MIGRATION-GUIDE.md
[^4]: Phaser README — build sizes (~1.29 MB minified, ~345 KB gzipped). https://github.com/phaserjs/phaser
[^5]: Phaser 3 release announcement. https://phaser.io/news/2018/02/phaser-3-released
[^6]: Phaser v4.2.1 changelog. https://github.com/phaserjs/phaser/blob/master/changelog/v4/4.2.1/CHANGELOG-v4.2.1.md

## Tags

javascript, typescript, game-development, html5, webgl, canvas, game-engine, 2d, browser, gamedev
