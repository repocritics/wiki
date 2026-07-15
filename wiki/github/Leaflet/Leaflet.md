# Leaflet/Leaflet

> The lightweight raster-and-marker mapping library that most web maps quietly run on.

[GitHub repo](https://github.com/Leaflet/Leaflet) ·
[Official website](https://leafletjs.com) ·
[License: BSD-2-Clause](https://github.com/Leaflet/Leaflet/blob/main/LICENSE)

## Overview

Leaflet is a JavaScript library for interactive maps, first released in 2011 by
Volodymyr Agafonkin[^1]. Its scope is deliberately narrow: it positions,
z-orders, and pans a stack of layers — raster tiles, markers, popups, and vector
shapes — over a slippy-map viewport, and gets out of the way. At roughly 40 kB
gzipped JS plus a few kB of CSS[^2], it is an order of magnitude smaller than the
WebGL vector-tile engines that came after it, and that size discipline is the
whole point.

The defining tension in 2026 is raster vs. vector. Leaflet's core renders raster
tiles (pre-rendered image tiles served over an XYZ/TMS scheme) and draws vectors
with SVG or Canvas — there is no built-in WebGL, no native vector-tile support,
and no client-side style engine. For pin-a-marker-on-a-basemap work this is a
feature: it runs everywhere, degrades gracefully, and has almost no conceptual
surface area. For 3D terrain, data-dense visualizations, or runtime restyling of
vector tiles, it is the wrong tool and a plugin will only get you part way.

Leaflet ships no data and no default tile server. You bring your own tile
provider (OpenStreetMap, a commercial basemap, or your own), which keeps the
library unopinionated but pushes the attribution, licensing, and rate-limit
questions onto you.

## Getting Started

```bash
npm install leaflet
# or use the CDN build directly in a <script>/<link> pair
```

```html
<!-- The CSS is mandatory — without it the map renders as broken tiles. -->
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
<div id="map" style="height: 400px"></div>   <!-- container MUST have a height -->
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<script>
  const map = L.map('map').setView([51.505, -0.09], 13);

  // No tiles ship with Leaflet — you supply a provider and its attribution.
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '&copy; OpenStreetMap contributors',
  }).addTo(map);

  L.marker([51.5, -0.09]).addTo(map)
    .bindPopup('A pin on the basemap.')
    .openPopup();
</script>
```

## Architecture / How It Works

Everything hangs off a central `L.Map` instance that owns the viewport, the
coordinate reference system, and a set of **panes** — stacked DOM containers that
enforce z-order (tiles below overlays below markers below popups).

- **Class system.** Leaflet predates ES classes and ships its own `L.Class` with
  `extend`/`include`/`Mixin`, and an `L.Evented` base for its `on`/`off`/`fire`
  event model. Every layer and control is an `L.Class` subclass, which is why the
  plugin ecosystem is large and consistent — extending Leaflet is the same
  mechanism the core uses.
- **Layers.** `L.GridLayer` is the base for tiled content; `L.TileLayer` loads
  XYZ raster tiles as `<img>` elements positioned absolutely and recycled as you
  pan. Markers are DOM elements. Vector shapes (`L.Polyline`, `L.Polygon`,
  `L.Circle`) are drawn by a pluggable renderer — `L.SVG` by default, `L.Canvas`
  when you have many features.
- **Coordinate systems.** Default CRS is `EPSG:3857` (Web/Spherical Mercator),
  matching mainstream tile providers. `EPSG:4326`, `EPSG:3395`, and a pixel-space
  `L.CRS.Simple` are built in; anything else (UTM, national grids) requires the
  Proj4Leaflet plugin.
- **Interaction.** Pan/zoom/tap/drag handlers are separate handler objects
  attached to the map, so behaviors can be individually disabled. Mobile touch,
  inertia, and tap handling are first-class — the "mobile-friendly" claim is
  structural, not marketing.

The coupling is loose by design: layers don't know about each other, only about
the map and their pane. That is what makes Leaflet easy to extend and hard to
make fast for very large datasets — there is no scene graph, no batching, and no
GPU path in core.

## Production Notes

- **Two footguns cause most "blank map" reports.** (1) The container needs an
  explicit height; a `<div>` with no height collapses to zero pixels and the map
  silently renders nothing. (2) `leaflet.css` must be loaded — miss it and tiles
  stack in a broken column. Both are FAQ #1 for a reason.
- **No bundled tile provider, and OSM's tiles are not a CDN.** The public
  `tile.openstreetmap.org` endpoint has a usage policy that forbids heavy or
  commercial traffic[^3]; shipping it to production will get you throttled. Use a
  paid provider or self-host tiles, and always honor the attribution requirement.
- **Marker scale.** Every marker is a DOM node. A few hundred is fine; a few
  thousand degrades pan/zoom noticeably. The standard fixes are the
  Leaflet.markercluster plugin or switching to the Canvas renderer / circle
  markers. There is no virtualized marker layer in core.
- **Vector tiles need a plugin, and it is a compromise.** Leaflet.VectorGrid can
  render vector tiles, but you do not get GPU rendering or runtime styling like a
  true vector engine — for that, reach for MapLibre GL instead of fighting
  Leaflet.
- **`invalidateSize()`.** A map created inside a hidden or later-resized
  container (tabs, modals, flex layouts) computes the wrong dimensions; call
  `map.invalidateSize()` after it becomes visible. This surprises nearly every
  team once.
- **The 2.0 transition is a real migration.** Leaflet 2.0 (in development on
  `main`) modernizes to ES modules and drops legacy-browser support, changing how
  the library is imported and consumed[^4]. Pin to the 1.9.x line if you rely on
  the global `L` and script-tag usage, and read the migration notes before
  upgrading rather than bumping the range blindly.
- **Framework wrappers add a layer to debug.** React-Leaflet, Vue-Leaflet, and
  similar bind Leaflet's imperative API to a declarative one; version-skew
  between the wrapper's peer range and Leaflet itself is a common source of
  breakage after upgrades.

## When to Use / When Not

**Use when:**
- You need a basemap with markers, popups, and a few shapes, and you want it
  small and working in every browser.
- You value a huge plugin ecosystem and a stable, readable API over raw rendering
  power.
- Your data is modest (hundreds to low thousands of features) and mostly raster.

**Avoid when:**
- You need vector tiles with client-side styling, 3D, terrain, or WebGL — use a
  GL engine.
- You're rendering tens of thousands of points or large geospatial datasets
  interactively.
- You need advanced GIS/projection handling beyond Web Mercator without leaning
  on plugins.

## Alternatives

- maplibre/maplibre-gl-js — use when you need GPU vector-tile rendering, runtime
  restyling, or 3D/terrain (the open fork of Mapbox GL).
- openlayers/openlayers — use when you need heavyweight GIS: many projections,
  WMS/WMTS, and advanced vector editing, and can accept a larger, steeper API.
- visgl/deck.gl — use when the map is really a large-scale data visualization
  (millions of points), often layered over a GL basemap.
- mapbox/mapbox-gl-js — use if you're already on the Mapbox platform and accept
  its non-open license terms.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2011-05 | First public release[^1]. |
| 1.0.0 | 2016-09 | First stable major; layer/renderer API settled[^5]. |
| 1.7.x | 2020 | Maintenance line; broad plugin compatibility. |
| 1.9.x | 2022–2023 | Current stable line (1.9.4 is the last 1.x)[^6]. |
| 2.0 | in dev | ES-module rewrite, drops legacy browsers, removes global `L`[^4]. |

## References

[^1]: Leaflet — project history and authorship, Volodymyr Agafonkin. https://leafletjs.com/
[^2]: Leaflet README — "about 40 kB of gzipped JS plus 3.2 kB of gzipped CSS." https://github.com/Leaflet/Leaflet/blob/main/README.md
[^3]: OpenStreetMap Foundation, "Tile Usage Policy." https://operations.osmfoundation.org/policies/tiles/
[^4]: Leaflet 2.0 development / announcements. https://leafletjs.com/2024/01/26/leaflet-2.0.0-alpha.html
[^5]: Leaflet blog, "Leaflet 1.0 final." https://leafletjs.com/2016/09/27/leaflet-1.0-final.html
[^6]: Leaflet GitHub releases. https://github.com/Leaflet/Leaflet/releases

## Tags

javascript, mapping, gis, web-maps, leaflet, raster-tiles, geospatial, frontend, browser, cartography, open-source
