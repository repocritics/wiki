# Turfjs/turf

> A modular, GeoJSON-native geospatial analysis toolkit for JavaScript and TypeScript — spatial ops, measurements, and transforms that run the same in the browser and Node.

[GitHub repo](https://github.com/Turfjs/turf) ·
[Official website](https://turfjs.org) ·
[License: MIT](https://github.com/Turfjs/turf/blob/master/LICENSE)

## Overview

Turf is a client- and server-side library for spatial analysis in JavaScript. Every function takes GeoJSON in and returns GeoJSON out, synchronously, with no external service, no native bindings, and no database[^1]. That design decision — pure JS operating directly on the format that web maps already speak — is why Turf became the default "do some geometry in the browser" library for the Mapbox/Leaflet/MapLibre ecosystem. It first appeared in 2013 and went through a defining rewrite around the v3 line that established the GeoJSON-in/GeoJSON-out contract it still uses[^2].

The library is deliberately modular. You can install the meta-package `@turf/turf` and get everything, or install exactly the functions you need as individual scoped packages (`@turf/distance`, `@turf/buffer`, `@turf/boolean-intersects`, …). Each function is its own npm package sharing a common `@turf/helpers` core. This granularity is the point: a bundle that only computes distances should not ship a polygon-clipping engine.

The central tension is accuracy versus convenience. Turf is not a projection-aware GIS engine. Measurement functions assume WGS84 longitude/latitude and model the Earth as a sphere; overlay/boolean functions treat those same degrees as a flat Cartesian plane. For visualization, UI interactions, and coarse analytics this is fine and fast. For survey-grade distances, correct areas at high latitude, or robust topology on projected data, Turf is the wrong tool and you will get quietly wrong numbers rather than errors.

## Getting Started

```bash
npm install @turf/turf          # everything
# or, pick individual functions to keep bundles small:
npm install @turf/distance @turf/buffer @turf/helpers
```

```js
import { point, distance, buffer } from "@turf/turf";

const from = point([-75.343, 39.984]);   // GeoJSON order is [lng, lat]
const to = point([-75.534, 39.123]);

const km = distance(from, to, { units: "kilometers" });   // ~97.16

// grow a 5 km ring around a point
const ring = buffer(from, 5, { units: "kilometers" });
```

Note the coordinate order: GeoJSON positions are `[longitude, latitude]`, the reverse of the `lat, lng` most mapping UIs display. This is the single most common Turf bug.

## Architecture / How It Works

Turf is a monorepo of ~100+ small packages, each exporting one function, built and versioned together (historically via Lerna)[^3]. There is no shared runtime object or spatial index; every call is a stateless transform over GeoJSON geometry. Internally the functions split into a few families with very different math:

- **Measurement** (`distance`, `area`, `along`, `bearing`, `length`, `midpoint`) — spherical trigonometry on lng/lat. Distances use the haversine formula against a mean Earth radius (~6,371 km); areas use a spherical-excess formula. These are geodesic approximations, not ellipsoidal (Vincenty/Karney) calculations, so expect sub-percent error on distances and larger relative error on small or high-latitude areas.
- **Boolean / overlay** (`union`, `intersect`, `difference`, `booleanIntersects`, `booleanContains`) — planar Cartesian geometry. `union`/`intersect`/`difference` delegate to the `polygon-clipping` library, which operates on the raw degree coordinates as if they were flat X/Y[^4]. This is fast and usually visually correct at city scale but distorts near the poles and breaks across the ±180° antimeridian.
- **Transformation** (`buffer`, `simplify`, `bboxClip`, `voronoi`) — a mix; `buffer` reprojects internally and leans on `jsts`, `simplify` uses Douglas–Peucker.
- **Classification / aggregation** (`nearestPoint`, `pointsWithinPolygon`, `collect`, `clustersKmeans`) — brute-force loops unless you supply your own index.

Because everything is GeoJSON in / GeoJSON out and side-effect free, composition is trivial and functions are individually tree-shakeable. The cost of that simplicity is that Turf carries no notion of a coordinate reference system: it cannot reproject, and it trusts you to hand it WGS84 degrees.

## Production Notes

- **Bundle size.** Importing from `@turf/turf` pulls the entire library (hundreds of KB minified) even if you use two functions. Bundlers can tree-shake named ESM imports, but the safe, predictable win is to depend on individual `@turf/*` packages. `buffer`, `intersect`, and `voronoi` are the heavy ones — they drag in `jsts`/`polygon-clipping`/`d3-voronoi`.
- **No spatial index by default.** `pointsWithinPolygon`, `nearestPoint`, `tag`, etc. are O(n) or O(n·m). For large FeatureCollections, build an index with `@turf/geojson-rbush` and query it yourself; naive Turf calls over 100k+ features will stall the main thread.
- **Single-threaded, main-thread blocking.** Pure JS with no worker offloading. Heavy overlays or buffers on big geometries freeze the browser tab. Move them to a Web Worker or the server.
- **Robustness of boolean ops.** `polygon-clipping` is sensitive to invalid input — self-intersecting rings, near-duplicate vertices, and floating-point coincidences can throw or return empty/garbage geometry. Clean and validate inputs (e.g. `@turf/kinks`, snapping) before `union`/`intersect`.
- **Antimeridian and poles.** Planar overlay math and spherical measurement math both misbehave around ±180° longitude and extreme latitudes. Turf does not split geometries at the antimeridian for you.
- **Winding order.** GeoJSON (RFC 7946) specifies right-hand-rule winding[^5]; some Turf functions are tolerant and some are not. `@turf/rewind` normalizes it when a third-party source gets it wrong.
- **Accuracy expectations.** If a stakeholder needs legally or scientifically defensible areas/lengths, do the math in a projected CRS with PostGIS/GDAL/Shapely and treat Turf as the interactive front-end approximation.

## When to Use / When Not

**Use when:**
- You need spatial ops in the browser next to a web map (hit-testing, buffers, measuring, drawing tools).
- Your data is already GeoJSON in WGS84 and you want zero-dependency, zero-service geometry.
- You want to install only the handful of functions a feature needs.
- Approximate, visualization-grade accuracy is acceptable.

**Avoid when:**
- You need projection-aware or ellipsoidal (survey-grade) measurements.
- You need OGC/JTS-grade topological robustness on complex or dirty geometry.
- You must reproject between coordinate systems — Turf can't, pair it with proj4.
- You're crunching very large datasets where a spatial database or native engine belongs on the server.

## Alternatives

- bjornharrtell/jsts — a JavaScript port of the JTS Topology Suite; use it instead when you need OGC-compliant, robust planar topology and predicates rather than convenience.
- proj4js/proj4js — use alongside or instead of Turf whenever real coordinate-system reprojection is the actual need.
- uber/h3-js — use instead when the problem is hierarchical hex-grid spatial indexing and aggregation, not ad-hoc geometry.
- d3/d3-geo — use instead when you need map projections and spherical geometry for rendering rather than analysis.
- libgeos/geos or Toblerity/Shapely (server side) — use instead when authoritative, validated geometry belongs in the backend/database.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013 | First release; modular geospatial functions for JS[^1]. |
| 3.x | 2016 | Defining rewrite — synchronous GeoJSON-in/GeoJSON-out contract[^2]. |
| 5.x | 2017 | Scoped `@turf/*` packages published from a single monorepo[^3]. |
| 6.x | 2019 | Source rewritten in TypeScript; first-party type definitions. |
| 7.x | 2024 | Dual ESM/CJS output and build modernization[^6]. |

## References

[^1]: Turf.js — official site and API docs. https://turfjs.org
[^2]: Turf.js documentation, "Getting Started". https://turfjs.org/docs/getting-started
[^3]: `@turf/turf` on npm — meta-package and modular package layout. https://www.npmjs.com/package/@turf/turf
[^4]: `polygon-clipping` — the planar overlay engine behind Turf's boolean operations. https://github.com/mfogel/polygon-clipping
[^5]: RFC 7946, "The GeoJSON Format" — coordinate order and winding rules. https://datatracker.ietf.org/doc/html/rfc7946
[^6]: Turf.js releases. https://github.com/Turfjs/turf/releases

## Tags

javascript, typescript, geospatial, gis, geojson, computational-geometry, spatial-analysis, mapping, browser, nodejs
