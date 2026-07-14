# plotly/plotly.py

> Python bindings that generate interactive, browser-rendered charts by emitting JSON specs for the plotly.js WebGL/SVG engine.

[GitHub repo](https://github.com/plotly/plotly.py) ·
[Official website](https://plotly.com/python/) ·
[License: MIT](https://github.com/plotly/plotly.py/blob/main/LICENSE.txt)

## Overview

plotly.py is the Python layer over plotly.js, the JavaScript charting engine Plotly, Inc. has developed since 2015[^1]. It does no rendering itself: every figure is ultimately a JSON structure (a tree of `data` traces and a `layout`) that gets handed to plotly.js, which draws it in a browser, notebook output cell, or exported image. The Python side is essentially a large, typed object model plus convenience constructors that build and validate that JSON. This is the defining fact about the library — understanding it explains most of its strengths and most of its footguns.

The audience is data scientists and analysts who want interactivity (hover tooltips, zoom, pan, legend toggling, 3D rotation) without writing JavaScript, and it is the charting substrate under Dash, Plotly's Python web-app framework. Since version 4 (2019) the package is fully open source and offline by default; the earlier requirement to authenticate against Plotly's hosted "Chart Studio" cloud was split out into a separate `chart-studio` package[^2].

The central tension is weight versus interactivity. A plotly figure carries its data inline as JSON and pulls in the ~3 MB plotly.js bundle to render. For a handful of interactive exploratory charts this is a fine trade; for a report with hundreds of figures or a print-oriented static output it is heavy compared to Matplotlib. The library has two API layers reflecting different eras: the verbose `graph_objects` model and the concise `plotly.express` wrapper added in 2019.

## Getting Started

```bash
pip install plotly
# static image (PNG/SVG/PDF) export also needs kaleido:
pip install -U kaleido
```

```python
import plotly.express as px

df = px.data.gapminder().query("year == 2007")

fig = px.scatter(
    df, x="gdpPercap", y="lifeExp",
    size="pop", color="continent", hover_name="country",
    log_x=True, size_max=60,
)
fig.update_layout(title="Life expectancy vs GDP, 2007")

fig.show()               # opens in browser / renders inline in a notebook
fig.write_html("out.html")   # standalone interactive file
fig.write_image("out.png")   # static export via kaleido
```

The same figure can be built with the lower-level object model, which `plotly.express` produces under the hood:

```python
import plotly.graph_objects as go

fig = go.Figure(
    data=[go.Scatter(x=[1, 2, 3], y=[4, 1, 2], mode="markers")],
    layout=go.Layout(title="manual"),
)
```

## Architecture / How It Works

A figure is a `plotly.graph_objects.Figure` whose entire state is a nested dict conforming to the plotly.js schema. That schema is not hand-written — it is generated from plotly.js's own `plot-schema.json`, and the thousands of Python trace/attribute classes (`go.Scatter`, `go.Bar`, `fig.layout.xaxis.tickfont`, ...) are codegenerated from it. This is why the Python API tracks plotly.js releases closely and why attribute names read like JavaScript (`marker.colorbar.tickvals`).

Two API layers sit on top of that model:

- **`plotly.graph_objects`** — explicit, validated construction. Every attribute assignment is checked against the schema at set time, which catches typos but makes the code verbose. This is the ground truth; everything else compiles down to it.
- **`plotly.express`** — a high-level, DataFrame-oriented facade added in mid-2019 (originally the separate `plotly_express` package, then folded in)[^3]. One function call maps tidy/long-form data plus column names to a fully configured figure. It absorbed the older `plotly.figure_factory` role for most common charts.

Rendering is pluggable through **renderers** (`plotly.io.renderers`). `fig.show()` picks a renderer from the environment: `notebook`/`jupyterlab`, `browser`, `vscode`, `colab`, `png`, `json`, and others. Getting a blank output almost always means the wrong renderer was auto-selected for the runtime.

Interactive display in classic notebooks and JupyterLab historically shipped as a separate `jupyterlab-plotly` / ipywidgets extension. As of the version-6 line the Jupyter integration was reworked around **anywidget**, reducing the extension-install friction that plagued earlier setups[^4].

**Static image export** does not use Python drawing. `kaleido` launches a headless Chromium that loads plotly.js, renders the figure, and screenshots it. This is the recommended path since plotly 4.9; the older `orca` Electron binary is legacy[^5]. Export therefore drags in a browser engine as a dependency — the single most common source of deployment surprise.

## Production Notes

**Bundle and payload weight.** Interactive output embeds the figure data as JSON and needs plotly.js (~3 MB minified) to render. `write_html(..., include_plotlyjs="cdn")` keeps the file small by loading the library from a CDN, at the cost of an external dependency and no offline viewing. `include_plotlyjs=True` inlines it for a self-contained but large file. For many figures on one page, load plotly.js once rather than per-figure.

**Static export is the deployment footgun.** `write_image` / `fig.to_image` require kaleido, which bundles Chromium. In slim Docker images, serverless functions, and CI you must install it explicitly and provide the shared libraries Chromium needs; a missing `libgobject`/`libnss3` produces cryptic failures. Note also the ecosystem churn: kaleido's v0.x and the later v1 line changed internals, and older plotly pinned specific kaleido versions — mismatches surface as export errors[^5].

**Large datasets.** SVG traces (`go.Scatter`) render each point as a DOM node and become sluggish past low tens of thousands of points. The `Scattergl` / WebGL trace family (`scattergl`, `scatterpolargl`, `pointcloud`) offloads to the GPU and handles far more points, but has subtle rendering differences and a per-page WebGL-context limit — too many gl traces on one page exhaust browser contexts. For very large data, downsample or aggregate server-side.

**Notebook rendering fragility.** Whether a figure appears depends on renderer selection, and static HTML exports of notebooks, email, and some CI viewers will show nothing interactive. When embedding, be explicit: set `pio.renderers.default` or export to HTML rather than relying on autodetection.

**API-layer confusion.** `plotly.express` returns `graph_objects` figures, so mixing them is normal and expected: build with `px`, then reach into `fig.update_traces(...)` / `fig.update_layout(...)` for anything Express doesn't expose. Beginners often get stuck looking for an Express keyword that doesn't exist when the answer is a graph-objects post-hoc update.

**Version boundaries.** The 3→4 jump (2019) removed the online/Chart Studio default and offline-by-default became the norm[^2]. The 5→6 line reworked Jupyter support around anywidget[^4] and continued dropping Python-2-era and legacy widget code paths. Pin `plotly` and its `kaleido` companion together in production requirements.

## When to Use / When Not

**Use when:**
- You want real interactivity (hover, zoom, 3D rotation, legend toggling) with no JavaScript.
- You are building a Dash app — plotly.py is the native figure type.
- You need 3D, WebGL scatter, maps/choropleths, or financial/statistical chart types out of the box.
- Output is a notebook or a web page where an interactive HTML figure is welcome.

**Avoid when:**
- You need publication-quality static/print figures with fine typographic control — Matplotlib gives more.
- Payload size or offline-without-CDN matters and you have many figures per page.
- You want a lightweight dependency tree — plotly plus kaleido plus a browser engine is heavy.
- You are rendering millions of points and even WebGL traces stall; pre-aggregate or use a specialized tool.

## Alternatives

- matplotlib/matplotlib — use instead when you need static, print-quality figures with exact control and a light dependency footprint.
- bokeh/bokeh — use instead when you want Python-driven browser interactivity with a server-side streaming/callback model rather than JSON-blob figures.
- altair-viz/altair — use instead when you prefer a concise declarative grammar (Vega-Lite) for statistical charts and don't need 3D/WebGL.
- holoviz/holoviews — use instead when you want a high-level layer that can target Bokeh, Matplotlib, or Plotly backends interchangeably.
- plotly/dash — not an alternative but the companion: use when the interactive figures need to live inside a full Python web application.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-11 | Repository created; Python client for the Plotly hosted service[^1]. |
| 3.0 | 2018-07 | Figure objects with in-place updates and validation; offline improvements. |
| 4.0 | 2019-07 | Offline by default; online/Chart Studio split into `chart-studio`[^2]. |
| 4.x | 2019 | `plotly.express` folded in from the standalone `plotly_express` package[^3]. |
| 4.9 | 2020 | `kaleido` becomes the recommended static-export engine; orca deprecated[^5]. |
| 5.0 | 2021-06 | Major line; dropped Python 2 remnants, reworked JupyterLab extension packaging. |
| 6.0 | 2025 | Jupyter integration re-based on anywidget; further legacy cleanup[^4]. |

## References

[^1]: Plotly, Inc. company and product history. https://plotly.com/python/ · repository created 2013-11-21 (GitHub API).
[^2]: plotly.py 4.0 release — offline by default, online features moved to the `chart-studio` package. https://github.com/plotly/plotly.py/blob/main/CHANGELOG.md
[^3]: Plotly, "Introducing plotly.express" (Plotly Express joins plotly.py). https://plotly.com/python/plotly-express/
[^4]: plotly.py Jupyter/anywidget integration notes. https://plotly.com/python/getting-started/
[^5]: plotly.py static image export docs (kaleido recommended since 4.9, orca legacy). https://plotly.com/python/static-image-export/

## Tags

python, data-visualization, charting, interactive, plotly, plotlyjs, webgl, jupyter, dashboard, declarative
