# bokeh/bokeh

> A Python plotting library that renders interactive graphics in the browser by shipping a serialized model to its own TypeScript runtime.

[GitHub repo](https://github.com/bokeh/bokeh) ·
[Official website](https://bokeh.org) ·
[License: BSD-3-Clause](https://github.com/bokeh/bokeh/blob/HEAD/LICENSE.txt)

## Overview

Bokeh is an interactive visualization library first released in 2013 and now a fiscally-sponsored project of NumFOCUS[^1]. You write Python; Bokeh builds an in-memory graph of plot objects (glyphs, axes, data sources, tools), serializes it to JSON, and hands it to **BokehJS**, a separate TypeScript rendering engine that draws to HTML canvas and wires up pan/zoom/hover/selection in the browser. This split is the defining fact of the project: the repository is majority TypeScript (the runtime) with a large Python package on top, and the two halves version together. At ~20.4k stars and ~4.3k forks with commits landing daily, it is one of the long-running, actively maintained pillars of the PyData plotting ecosystem alongside matplotlib and Plotly.

The central tension is level of abstraction. Bokeh's model API is lower-level and more explicit than most competitors — you assemble figures from primitives, which buys fine control over interactivity but costs verbosity for routine charts. Much of the ecosystem papers over this: **HoloViews** and **hvPlot** provide terse high-level APIs that emit Bokeh, and **Panel** builds full dashboards and apps on the same runtime. In practice many teams reach for Bokeh indirectly through those layers rather than writing `bokeh.plotting` calls by hand.

The second tension is the two rendering paths. Standalone output (HTML/JSON embedded in a notebook or page) needs no running server. The **Bokeh server** path keeps the object model live in a Python process and syncs it to the browser over a WebSocket, which is what enables Python callbacks on widget events — and also what turns a "plot" into a stateful web service you have to deploy and scale.

## Getting Started

```bash
pip install bokeh      # or: conda install bokeh
```

```python
from bokeh.plotting import figure, show
from bokeh.models import HoverTool

p = figure(title="Sine", width=600, height=300,
           x_axis_label="x", y_axis_label="sin(x)")

x = [i / 20 for i in range(100)]
y = [__import__("math").sin(v) for v in x]

p.line(x, y, line_width=2, legend_label="sin(x)")
p.add_tools(HoverTool(tooltips=[("x", "@x"), ("y", "@y")]))

show(p)   # writes a self-contained HTML file and opens it
```

For interactive apps with Python callbacks, you write a script and run it under the server: `bokeh serve --show app.py`.

## Architecture / How It Works

Bokeh is two codebases pretending to be one library:

1. **Python (`bokeh`)** — a declarative object model. Every plot element is a `Model` subclass with typed, validated properties. `bokeh.plotting.figure` is a convenience layer over `bokeh.models`. Building a figure mutates a document graph; it does not draw anything.
2. **BokehJS (`bokehjs/`, TypeScript)** — the renderer. It receives the serialized document, reconstructs the model graph client-side, and paints glyphs to canvas (with a WebGL path for large scatter/line data). It owns all interaction: hover, box-select, wheel-zoom, linked brushing.

The contract between them is a JSON serialization format. This is why the repo's primary language reads as TypeScript despite being "a Python library": the runtime that users never see directly is the larger half, and every new glyph or tool must be implemented on both sides.

`ColumnDataSource` is the load-bearing abstraction. It is a columnar table shared by reference across glyphs; updating its `.data` is how you drive streaming updates and linked selection. Selections and ranges are themselves models, so "when the user selects points here, filter the table there" is expressed as shared model state rather than event plumbing.

The **server** promotes the document from a one-shot export to a synchronized session. Each connected browser gets a `Document` backed by a Python `Session`; property changes propagate over a Tornado WebSocket in both directions. Python callbacks (`on_change`, `on_click`) run in the server process. This is genuinely different from Plotly Dash's request/response callback model — Bokeh keeps a stateful bidirectional channel open per session.

## Production Notes

- **Standalone vs. server is a deployment fork, not a toggle.** Static HTML/notebook output scales trivially (it's a file). The server is a stateful Tornado app holding a live session per browser tab — memory grows with concurrent users, and you need sticky sessions behind a load balancer because state lives in one process. Plan for horizontal scaling and session eviction from the start if you go the server route.
- **Version lock between Python and JS.** The Python `bokeh` and the embedded BokehJS must match. Mixed versions (e.g. a cached CDN BokehJS against a newer Python export) produce silent render failures or deserialization errors. Pin exact versions and, for offline use, bundle resources with `INLINE` rather than the default CDN.
- **3.0 was a breaking rewrite.** The 3.0 line (2022) reorganized modules and changed defaults; `bokeh.plotting` glyph method signatures shifted toward keyword-explicit forms and several `bokeh.models` imports moved. Code and Stack Overflow answers written for 1.x/2.x frequently do not run unmodified. The current maintained line is 3.x (default branch `branch-3.10`).
- **Large data needs the right escape hatch.** Rendering hundreds of thousands of glyphs in canvas is slow; use `output_backend="webgl"` for dense scatter/line, and for millions of points push server-side rasterization via **Datashader** (image the data, ship pixels, not points) rather than sending raw arrays to the browser.
- **Widget callbacks split into two worlds.** `CustomJS` callbacks run in the browser with no Python round-trip (works in static exports); Python callbacks require the server. Choosing the wrong one early forces a rewrite when you discover a static export can't call back into Python.
- **Notebook state is fiddly.** `output_notebook()` must be re-run per kernel session, and re-executing cells can duplicate or orphan output; classic Jupyter, JupyterLab, and VS Code notebooks each have their own quirks with the BokehJS extension.

## When to Use / When Not

**Use when:**
- You need genuine browser interactivity (linked brushing, live selection, streaming updates) driven from Python.
- You want to build a data application or dashboard, especially via Panel, and want Python callbacks on the server.
- You need fine, explicit control over plot construction and are willing to trade brevity for it.
- You're plotting large datasets and can pair it with Datashader/WebGL.

**Avoid when:**
- You want a one-liner static chart — matplotlib or Plotly Express are terser for that.
- You prefer a declarative grammar-of-graphics style — Altair/Vega-Lite fits that mental model better.
- You need publication-quality static print figures (PDF/vector) — Bokeh is canvas-first and export-to-vector is limited.
- You want to avoid running a server but still need Python-side interactivity — that combination is exactly what Bokeh can't give you.

## Alternatives

- plotly/plotly.py — similar Python-to-browser interactivity with a denser high-level API (Plotly Express) and Dash for apps; use it when you want less boilerplate and a request/response callback model.
- matplotlib/matplotlib — use for static, publication-grade, vector-output figures where interactivity is not the point.
- vega/altair — use when you prefer a declarative grammar-of-graphics API (Vega-Lite) over imperative figure assembly.
- holoviz/holoviews — higher-level API that emits Bokeh; use it when you like the Bokeh runtime but want far less code.
- holoviz/panel — use when the real goal is a full dashboard/app rather than a single plot.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013 | Initial public release; Python-to-browser plotting concept established[^1]. |
| 1.0 | 2018-10 | First stable API milestone after years of pre-1.0 iteration. |
| 2.0 | 2020-03 | Dropped Python 2; layout and API cleanups. |
| 3.0 | 2022-09 | Major rewrite: module reorg, changed defaults, BokehJS overhaul[^2]. |
| 3.x | 2023–2026 | Ongoing 3.x line; current development on `branch-3.10`[^3]. |

## References

[^1]: Bokeh project / NumFOCUS sponsorship and history. https://bokeh.org and https://numfocus.org/project/bokeh
[^2]: Bokeh 3.0 release announcement and migration notes. https://docs.bokeh.org/en/latest/docs/releases.html
[^3]: Repository metadata (default branch `branch-3.10`, last push 2026-07-14), GitHub API `repos/bokeh/bokeh`, retrieved 2026-07-15. https://github.com/bokeh/bokeh

## Tags

python, typescript, data-visualization, interactive-plots, plotting, dashboards, jupyter, bokehjs, webgl, numfocus
