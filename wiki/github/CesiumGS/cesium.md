# CesiumGS/cesium

> A WebGL engine for streaming and rendering planet-scale 3D geospatial data in the browser, and the reference implementation of the 3D Tiles standard.

[GitHub repo](https://github.com/CesiumGS/cesium) ·
[Official website](https://cesium.com/cesiumjs/) ·
[License: Apache-2.0](https://github.com/CesiumGS/cesium/blob/main/LICENSE.md)

## Overview

CesiumJS is a JavaScript library for rendering a high-precision WGS84 globe and 2D/2.5D maps with WebGL, without browser plugins. It was open-sourced by Analytical Graphics, Inc. (AGI) in 2012 and is now developed by Cesium GS, Inc.[^1] Its distinguishing concern is scale and geodetic accuracy: it handles centimeter-level positioning on an ellipsoidal Earth, time-dynamic data, and terrain/imagery/3D-model datasets far larger than memory through streaming and level-of-detail. The audience is geospatial, aerospace, defense, AEC, and simulation developers, not general web-graphics users who would reach for three.js.

The project is the canonical engine behind **3D Tiles**, an open specification (now an OGC Community Standard) for streaming massive heterogeneous 3D content — photogrammetry, BIM, point clouds, instanced trees[^2]. It is also the reference renderer for **CZML** (a JSON format for time-dynamic scenes) and a heavy consumer of **glTF** for individual models.

The defining tension is open-core. CesiumJS the runtime is Apache-2.0 and genuinely usable standalone, but it is engineered as the client for **Cesium ion**, the company's commercial tiling/hosting SaaS. You can bring your own content and providers, but the smoothest path — and the default in most tutorials — routes through ion with an access token, and much of the world terrain/imagery tooling lives on the paid side[^3].

## Getting Started

```sh
npm install cesium --save
```

```js
import { Viewer, Cartesian3, Ion, createWorldTerrainAsync } from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";

// Required to stream Cesium ion assets; omit if using only your own providers.
Ion.defaultAccessToken = "<YOUR_ION_TOKEN>";

const viewer = new Viewer("cesiumContainer", {
  terrainProvider: await createWorldTerrainAsync(),
});

viewer.camera.flyTo({
  destination: Cartesian3.fromDegrees(-122.4175, 37.655, 400),
});
```

CesiumJS also ships as scoped packages — `@cesium/engine` (core, rendering, data APIs) and `@cesium/widgets` (the `Viewer` UI) — for finer dependency control[^4]. The bundler must be told where to find Cesium's static assets (workers, shaders, `Assets/`); most setups use `copy-webpack-plugin`, `vite-plugin-cesium`, or set `CESIUM_BASE_URL`.

## Architecture / How It Works

CesiumJS exposes two layered APIs over one renderer:

1. **Entity API** — a declarative, retained-mode layer. You add `Entity` objects with time-varying `Property` values; the `DataSource` system (CZML, GeoJSON, KML) feeds it. Good for data visualization, driven by a clock.
2. **Primitive API** — the low-level graphics layer. `Primitive`, `Geometry`, `Appearance`, and `Cesium3DTileset` sit close to WebGL. More control, more footguns.

Underneath, the core problem is that IEEE-754 32-bit floats (what GPUs use) cannot represent Earth-sized coordinates without visible jitter. Cesium solves this with **RTC (relative-to-center) encoding** and CPU-side high/low float splitting so geometry stays precise near the camera. Positions are carried in an Earth-Centered Earth-Fixed (ECEF) `Cartesian3` frame, not lat/long, internally.

Streaming is the other half. `Cesium3DTileset` loads a tileset JSON describing a bounding-volume hierarchy, then fetches tiles on demand based on **screen-space error** — a tile refines only when its projected error exceeds a threshold. This is what lets a city-scale photogrammetry model render on a laptop. Terrain uses quantized-mesh tiles; imagery is composited from tiled providers per zoom level.

Rendering runs a custom scene graph and a hand-written shader system (recently migrated toward a unified `Model` architecture that treats all 3D content as glTF under the hood). The engine is single-threaded on the main draw loop but offloads decoding (Draco, glTF, terrain) to Web Workers.

## Production Notes

- **The ion token is a runtime dependency, not a build one.** Default terrain, world imagery, and the geocoder call ion and will silently fail or rate-limit if the token is missing, expired, or over quota. Air-gapped/offline deployments must swap in self-hosted providers and follow the Offline Guide[^5].
- **Asset serving is a recurring setup failure.** Cesium loads Web Workers, WASM (Draco/Basis), and shader files at runtime relative to `CESIUM_BASE_URL`. Misconfiguring this yields 404s for workers that surface as blank globes rather than clear errors. Each bundler needs its own copy/config recipe.
- **Bundle size is large.** The full `cesium` package is multiple megabytes even minified; tree-shaking via `@cesium/engine` and importing individual modules helps but does not make it small. It is not appropriate for weight-sensitive pages.
- **Memory and GPU pressure scale with tileset size.** `maximumScreenSpaceError`, `cacheBytes`, and `maximumCacheOverflowBytes` on `Cesium3DTileset` are the main levers; leaving defaults on huge datasets can exhaust GPU memory or thrash the tile cache. Point clouds and dense photogrammetry are the usual offenders.
- **Async API migration.** Provider constructors moved from sync `new` to `...Async()` factory functions across the 1.10x releases; older tutorials using `new CesiumTerrainProvider(...)` no longer compile against current versions. Check the CHANGES.md for the version you pin.
- **Version cadence is roughly monthly**, with breaking changes flagged in CHANGES.md. Pin exact versions; minor bumps have removed deprecated APIs on a documented but brisk schedule.

## When to Use / When Not

**Use when:**
- You need an accurate WGS84 globe with real terrain, imagery, and time-dynamic data.
- You are consuming 3D Tiles, CZML, quantized-mesh terrain, or large photogrammetry/point-cloud datasets.
- Geodetic precision, coordinate transforms, and planet-scale streaming matter more than bundle size.
- You are building GIS, aerospace, defense, simulation, or digital-twin visualization.

**Avoid when:**
- You want a lightweight or artistic 3D scene — three.js or Babylon.js are smaller and more general.
- You need a 2D slippy map of tiles and markers — MapLibre/Leaflet are far lighter.
- You cannot tolerate a large bundle or a runtime dependency on an external asset provider.
- Your data is flat/local and never needs an ellipsoidal Earth or streaming LOD.

## Alternatives

- CesiumGS/cesium-native — use when you need the same 3D Tiles engine in C++/Unreal/Unity/O3DE rather than the browser.
- MapLibre/maplibre-gl-js — use for lightweight 2D/2.5D vector-tile web maps without a full 3D globe.
- mrdoob/three.js — use for general-purpose WebGL 3D where geospatial precision and streaming are not needed.
- visgl/deck.gl — use for large-scale data-overlay visualization layered on a map, often paired with a basemap rather than replacing it.
- OSGeo/Cesium-alternatives aside, iTowns (itowns/itowns) — use when you want an open 3D geospatial framework with tighter French IGN/OGC integration and a lighter footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| Open-source release | 2012-03 | AGI open-sources CesiumJS on GitHub[^1]. |
| 1.0 | 2014-08 | First stable 1.x; API considered production-ready. |
| 3D Tiles announced | 2016 | Streaming spec for massive 3D content, later an OGC standard[^2]. |
| 1.x async providers | ~2023 | Provider construction moved to `...Async()` factories. |
| Modular packages | 2022-12 | Split into `@cesium/engine` and `@cesium/widgets`[^4]. |
| Ongoing | monthly | Unified glTF-based `Model` architecture; regular CHANGES.md releases. |

## References

[^1]: Cesium history and stewardship (AGI → Cesium GS, Inc.). https://cesium.com/about/
[^2]: 3D Tiles specification, OGC Community Standard. https://github.com/CesiumGS/3d-tiles
[^3]: Cesium open-core business model. https://cesium.com/why-cesium/open-ecosystem/cesium-business-model/
[^4]: "Modular structure in CesiumJS" — Cesium blog, 2022-12-07. https://cesium.com/blog/2022/12/07/modular-structure-in-cesiumjs/
[^5]: CesiumJS Offline Guide. https://github.com/CesiumGS/cesium/blob/main/Documentation/OfflineGuide/README.md

## Tags

javascript, webgl, 3d-globe, geospatial, gis, 3d-tiles, gltf, czml, mapping, streaming, digital-twin
