# python-visualization/folium

> Python bindings that turn pandas/GeoPandas data into a Leaflet.js web map by generating a self-contained HTML document.

[GitHub repo](https://github.com/python-visualization/folium) ·
[Official website](https://python-visualization.github.io/folium/) ·
[License: MIT](https://github.com/python-visualization/folium/blob/main/LICENSE.txt)

## Overview

folium is a Python library that builds interactive [Leaflet.js](https://leafletjs.com) maps from Python objects. You describe a map — tile layer, markers, GeoJSON overlays, choropleths — in Python, and folium emits an HTML file with the corresponding Leaflet JavaScript baked in. It was created by Rob Story in 2013 and is now maintained by the community `python-visualization` organization[^1]. It remains pre-1.0; the current line is v0.20 (2025)[^2].

The defining characteristic — and the source of most confusion — is that folium is a *code generator*, not a rendering engine or a Jupyter widget. It produces a static HTML/JS artifact. There is no live channel back to the Python kernel: once the map is written, clicking it runs Leaflet in the browser, and Python never hears about it. This makes folium excellent for reports, dashboards embedded as HTML, and notebook output, but unsuitable when you need Python to react to map events (draw a polygon, get the coordinates back into a variable). That bidirectional use case belongs to `ipyleaflet` instead.

Because every visual element becomes a DOM node in the generated page, folium's practical ceiling is set by the *browser*, not by Python. A few thousand individual markers is where naive usage starts to stutter, and the standard answer is clustering or WebGL plugins rather than raw layers.

## Getting Started

```bash
pip install folium
# or
conda install -c conda-forge folium
```

```python
import folium

# center the map, pick a basemap
m = folium.Map(location=[45.5236, -122.6750], zoom_start=13,
               tiles="OpenStreetMap")

# add a marker with a popup
folium.Marker(
    [45.5236, -122.6750],
    popup="Portland, OR",
    tooltip="Click me",
    icon=folium.Icon(color="green"),
).add_to(m)

# write a standalone HTML file (opens in any browser)
m.save("map.html")

# in a Jupyter notebook, just return `m` in the last cell to render inline
m
```

The `add_to()` / `add_child()` pattern is the core idiom: every feature is an object you attach to the `Map` (or to a `FeatureGroup`). In Jupyter, `Map._repr_html_` renders the map inside an `srcdoc` iframe.

## Architecture / How It Works

folium is a thin Python layer over three pieces:

- **branca** — a sibling library, split out of folium, that provides the base `Element` / `Figure` / `MacroElement` tree and the Jinja2 templating machinery, plus colormaps and the HTML `<div>` scaffolding. Most folium objects are branca `MacroElement` subclasses carrying a Jinja2 `_template`[^3].
- **Jinja2** — each feature (marker, GeoJSON layer, tile layer, plugin) owns a template fragment. Rendering walks the element tree and concatenates the fragments into one HTML document: a `<head>` of CDN `<link>`/`<script>` tags for Leaflet and plugin assets, and a `<body>`/`<script>` that instantiates the Leaflet objects.
- **Leaflet.js** — the actual map runtime, pulled from a CDN at view time. folium pins specific Leaflet and plugin versions in its templates; upgrading Leaflet is a folium-side change, not something you control at runtime.

The output is deliberately self-contained *except* for CDN assets: the generated HTML references Leaflet, plugin JS/CSS, and tile servers over the network. It renders offline only if those assets and tiles are cached or vendored.

`folium.plugins` is a large submodule wrapping popular Leaflet plugins — `MarkerCluster`, `FastMarkerCluster`, `HeatMap`, `Draw`, `Fullscreen`, `TimestampedGeoJson`, `Geocoder`, `MousePosition`, and others. Each is another `MacroElement` with its own CDN dependency and template. `Choropleth` and `GeoJson` accept GeoJSON dicts, file paths, or GeoPandas `GeoDataFrame`s and serialize the geometry inline into the page.

Because geometry and every marker are embedded as literal JSON/JS in the document, the HTML file size scales linearly with feature count — a choropleth of detailed polygons or thousands of points produces a multi-megabyte page.

## Production Notes

- **Feature count is the real limit.** Individual `Marker`/`CircleMarker` objects each become Leaflet layers and DOM nodes. Past a few thousand, browser interaction degrades. Use `MarkerCluster` / `FastMarkerCluster` for point data, or a WebGL layer (`folium-glify-layer`, vector tiles) for large GeoJSON. `FastMarkerCluster` takes raw coordinate lists and skips per-marker Python object creation, which also cuts generation time.
- **Page weight.** Since data is inlined, simplify polygon geometry before handing it to `GeoJson`/`Choropleth` (e.g. `shapely` / `mapshaper` / `topojson`), or the saved HTML balloons. There is no server-side tiling of your data.
- **No callbacks to Python.** Map interactions do not return to the kernel. If you need to capture a drawn shape or a clicked coordinate in Python, folium alone cannot do it — reach for `ipyleaflet` (a real Jupyter widget) or `streamlit-folium`, which surfaces some interaction state back into a Streamlit app.
- **Tile provider terms.** The default OpenStreetMap tiles and other providers have usage policies and attribution requirements; heavy or commercial use of a public tile server will get rate-limited. Many named tilesets require an API key, increasingly sourced via `xyzservices`.
- **Rendering surfaces differ.** Inline rendering relies on `_repr_html_` and an iframe; some environments (certain nbconvert/exported-HTML, email, restricted CSP contexts) block the iframe or the CDN scripts and show a blank map. For fixed exports, `m.save()` to HTML is the reliable path.
- **Pre-1.0 API churn.** Being <1.0, minor releases have removed or renamed things (older code often breaks on `folium.Map` keyword changes and plugin moves). Pin the version; read release notes before upgrading[^2].

## When to Use / When Not

**Use when:**
- You have tabular or geospatial data in pandas/GeoPandas and want an interactive map with minimal code.
- The deliverable is a report, notebook, or embeddable HTML page — one-directional display.
- You want choropleths, clustered points, or heatmaps without writing JavaScript.

**Avoid when:**
- You need map events back in Python (draw-to-select, live coordinate capture) — use `ipyleaflet`.
- You're plotting hundreds of thousands of points without aggregation — use a WebGL/datashading stack (`pydeck`, `datashader`, `lonboard`).
- You need publication-quality static/vector maps for print — use `geopandas.plot` / `matplotlib` / `cartopy`.
- You want a fully offline artifact with no CDN dependency — folium's templates fetch Leaflet and tiles over the network.

## Alternatives

- `jupyter-widgets/ipyleaflet` — bidirectional Leaflet as a real Jupyter widget; use it when Python must react to map interactions.
- `visgl/deck.gl` (via `pydeck`) — use for GPU-rendered large point/arc/hexbin layers folium can't handle.
- `geopandas/geopandas` — use `GeoDataFrame.plot()` / `.explore()` when you want static matplotlib maps (or note that `.explore()` is itself folium under the hood).
- `holoviz/hvplot` + `datashader` — use for millions of points via server-side rasterization.
- `developmentseed/lonboard` — use for fast Arrow-backed GeoArrow rendering of large vector data in notebooks.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2013–2014 | Initial releases by Rob Story; Leaflet wrapper. |
| 0.2.0 | 2015-10 | Early API, IPython/Jupyter inline display. |
| 0.5.0 | 2017-06 | Modernized API on the branca element/template model[^3]. |
| 0.16.0 | 2023 | Release-notes migrated to GitHub Releases[^1]. |
| 0.19.0 | 2024-12 | Continued plugin and typing improvements. |
| 0.20.0 | 2025-06-16 | Current line[^2]. |

## References

[^1]: folium README and repository, python-visualization/folium. https://github.com/python-visualization/folium
[^2]: folium releases (v0.20.0, 2025-06-16). https://github.com/python-visualization/folium/releases
[^3]: branca — the element/template and colormap library split out of folium. https://github.com/python-visualization/branca

## Tags

python, geospatial, mapping, leaflet, data-visualization, choropleth, jupyter, gis, html-generation, pandas
