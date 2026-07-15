# maplibre/maplibre-gl-js

> GPU-accelerated vector-tile map rendering in the browser — the community-governed OSS fork of mapbox-gl-js 1.x.

[GitHub repo](https://github.com/maplibre/maplibre-gl-js) ·
[Official website](https://maplibre.org) ·
[License: BSD-3-Clause](https://github.com/maplibre/maplibre-gl-js/blob/main/LICENSE.txt)

## Overview

MapLibre GL JS is a WebGL-based library that renders interactive vector-tile maps in the browser. It parses a style document (sources + layers), decodes vector tiles in Web Workers, and issues GPU draw calls for smooth pan/zoom/rotate/pitch. It is the reference implementation of the MapLibre Style Spec and the anchor project of the broader MapLibre org (which also maintains Native/iOS/Android bindings and a shared style spec).

The project exists because of a licensing rupture: in December 2020 Mapbox relicensed mapbox-gl-js away from open source at version 2.0[^1]. MapLibre GL JS was forked from the last BSD-3 release (mapbox-gl-js 1.x) and the 1.x line was intended as a drop-in replacement[^2]. That "drop-in" framing no longer holds — the two codebases have diverged substantially since, and code cannot legally be backported from post-1.x mapbox-gl-js because that code is not under the BSD-3 license[^3].

The defining tension is that MapLibre GL JS renders maps but does not provide the data. Unlike a hosted SDK with a bundled account, you must supply your own vector tiles and style (self-hosted, MapTiler, Protomaps PMTiles, Stadia, etc.). That makes it genuinely vendor-neutral, at the cost of a non-trivial infrastructure decision before the first map appears.

## Getting Started

```bash
npm install maplibre-gl
```

```html
<link href="https://unpkg.com/maplibre-gl@latest/dist/maplibre-gl.css" rel="stylesheet" />
<div id="map" style="width: 400px; height: 300px;"></div>
<script type="module">
import maplibregl from "https://unpkg.com/maplibre-gl@latest/dist/maplibre-gl.mjs";

const map = new maplibregl.Map({
  container: "map",
  style: "https://demotiles.maplibre.org/style.json", // your style + tile source
  center: [-74.5, 40], // [lng, lat]
  zoom: 9,
});
</script>
```

The CSS file is not optional — controls, popups, and marker positioning break without it. The `style` URL is the load-bearing input: `demotiles.maplibre.org` is a low-detail demo only; a real app points `style` at a provider's style JSON or a self-hosted one.

## Architecture / How It Works

The runtime is a main thread plus a pool of Web Workers:

- **Style** — a JSON document (MapLibre Style Spec, forked from the Mapbox GL Style Spec) declaring `sources` and `layers`. Layer paint/layout properties are driven by **expressions**, a small JSON DSL evaluated per-feature (`["get", "population"]`, `["interpolate", ...]`). This is the data-driven-styling model that separates MapLibre from raster tile libraries.
- **Sources** — vector (Mapbox Vector Tile / protobuf), raster, raster-dem (elevation, for terrain), GeoJSON, image, video. Vector and GeoJSON sources are tiled and dispatched to workers.
- **Workers** — decode MVT protobuf, run the style layers over features, and bucket geometry into typed arrays ready for the GPU. This keeps parse/layout work off the main thread; jank usually traces to worker saturation, not draw calls.
- **Render** — the main thread uploads buffers and issues WebGL draw calls each frame. Projection was Web Mercator (EPSG:3857) for most of the project's life; a real globe projection was added in the 5.x line[^4].

The library is intentionally headless about hosting: it speaks tile URLs and style JSON, not accounts or API keys. Terrain (3D), hillshade, and sky/atmosphere are additional render passes layered on the same source/worker machinery. WebGL2 is now the rendering target (see the repo's `webgl2` topic), which dropped some legacy-browser support along the way.

## Production Notes

**You must bring tiles and a style.** The most common first-project mistake is expecting maps to "just work." Options: MapTiler (hosted, API key), Protomaps PMTiles (single static file, no tile server — pairs well with MapLibre), Stadia/Stamen, or self-hosting with tileserver-gl, Martin, or tegola. Budget for tile hosting cost or CDN egress.

**Bundle size is large.** MapLibre GL JS is a heavyweight dependency (hundreds of KB gzipped). It is not a fit for pages that need a tiny footprint; Leaflet is an order of magnitude smaller if you only need raster tiles.

**Migration from mapbox-gl-js is not free.** Parity holds for the 1.x era API surface; anything relying on mapbox-gl-js 2.x+ features, or on Mapbox-hosted styles/tiles, requires rework. `react-map-gl` supports both but you configure the underlying library explicitly. Do not copy code from current mapbox-gl-js into MapLibre — it is a license violation and the maintainers treat unauthorized backports as an existential risk to the project[^3].

**RTL text.** Arabic/Hebrew and other right-to-left scripts need the RTL text plugin loaded explicitly (`setRTLTextPlugin`); without it, labels render in the wrong direction.

**GPU and context loss.** Terrain and globe are GPU-intensive and can struggle on low-end/integrated hardware. WebGL context loss (tab backgrounding, driver resets) must be handled; long-lived maps in dashboards should listen for and recover from it.

**Many layers/sources cost memory.** Each source keeps tiles cached; large numbers of vector layers, high `maxzoom`, or many simultaneous sources raise memory and worker load. Profile with the browser's performance tools when pans stutter — it is usually decode/layout in workers, not the frame loop.

## When to Use / When Not

**Use when:**
- You want vendor-neutral, GPU-accelerated vector maps with data-driven styling and no mandatory account.
- You need pan/zoom/rotate/pitch, 3D terrain, or a globe in a web app.
- You want to self-host the full stack (tiles + style + renderer) for cost or sovereignty reasons.
- You're standardizing on the MapLibre Style Spec across web, iOS, and Android.

**Avoid when:**
- You only need a simple raster tile map with a minimal bundle — Leaflet is lighter and simpler.
- You need heavy OGC/GIS features, arbitrary projections, or raster analysis — OpenLayers is more complete.
- You want a turnkey hosted SDK with tiles, geocoding, and routing bundled behind one account — that is Mapbox's or Google's model.
- Your visualization is large-scale data overlay first, basemap second — reach for deck.gl (often layered on top of MapLibre).

## Alternatives

- mapbox/mapbox-gl-js — the upstream this was forked from; use it when you're committed to Mapbox's hosted ecosystem and accept the non-OSS license and account/token requirement.
- openlayers/openlayers — use when you need many projections, OGC standards, or raster GIS features more than GPU vector-tile rendering.
- Leaflet/Leaflet — use for lightweight raster tile maps where a small bundle matters more than vector styling.
- visgl/deck.gl — use when the primary job is large-scale WebGL data visualization; commonly paired with MapLibre as the basemap.
- CesiumGS/cesium — use when you need a true 3D globe with terrain and geospatial precision beyond a web map.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-12 | Mapbox relicenses mapbox-gl-js at 2.0; community forks the BSD-3 1.x line[^1]. |
| 1.0 | 2021 | First MapLibre GL JS release, drop-in replacement for mapbox-gl-js 1.x[^2]. |
| 2.0 | 2022-02 | 3D terrain; dropped legacy browser (IE) support; TypeScript migration underway[^4]. |
| 3.0 | 2023 | Sky/atmosphere and rendering improvements; new project branding[^4]. |
| 4.0 | 2024 | WebGL2 rendering target; further spec and performance work[^4]. |
| 5.0 | 2025 | Globe projection (vertical perspective) added[^4]. |

## References

[^1]: Mapbox, mapbox-gl-js v2.0 license change (December 2020). https://github.com/mapbox/mapbox-gl-js/blob/main/CHANGELOG.md
[^2]: MapLibre GL JS README — "originated as an open-source fork of mapbox-gl-js, before their switch to a non-OSS license in December 2020." https://github.com/maplibre/maplibre-gl-js
[^3]: MapLibre GL JS README — backport policy: "Unauthorized backports are the biggest threat to the MapLibre project." https://github.com/maplibre/maplibre-gl-js
[^4]: MapLibre GL JS releases and changelog. https://github.com/maplibre/maplibre-gl-js/releases
[^5]: License note — GitHub's license detector reports NOASSERTION for this repo, but LICENSE.txt and the README both state the 3-Clause BSD license. https://github.com/maplibre/maplibre-gl-js/blob/main/LICENSE.txt

## Tags

typescript, javascript, webgl, vector-tiles, maps, geospatial, cartography, mapbox-gl-fork, gis, browser
