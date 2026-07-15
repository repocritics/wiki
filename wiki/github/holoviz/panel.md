# holoviz/panel

> Build dashboards and web apps entirely in Python, on top of Bokeh's server and the Param reactive object model.

[GitHub repo](https://github.com/holoviz/panel) ·
[Official website](https://panel.holoviz.org) ·
[License: BSD-3-Clause](https://github.com/holoviz/panel/blob/main/LICENSE.txt)

## Overview

Panel is a Python app framework for turning analysis code — plots, tables, models, widgets — into interactive dashboards and multi-page web apps without leaving Python[^1]. It is part of the HoloViz ecosystem (alongside hvPlot, HoloViews, Datashader, and Param) and originated inside the PyViz/Anaconda dataviz effort. Its stated selling point is breadth of plotting-library support: Panel wraps Matplotlib, Plotly, Bokeh, Altair/Vega, Deck.gl, ECharts, PyVista/VTK, Folium, and the ipywidgets ecosystem as first-class panes, rather than betting on one rendering stack.

Panel's defining architectural choice is that it is built on two lower-level libraries: **Bokeh** (for the server, the WebSocket transport, and the browser-side model synchronization) and **Param** (for the declarative, reactive object model that every widget and pane is an instance of). This is the source of both its power and its friction. Because components are Param objects, you get real bidirectional state, dependency tracking, and the ability to reuse the same code in a notebook, a served app, or a batch report. Because the transport is Bokeh's, you also inherit Bokeh's stateful-server model, its version coupling, and a learning curve steeper than single-file frameworks like Streamlit.

The practical tension: Panel scales *up* better than most Python app frameworks (complex multi-page apps, custom components, fine-grained callbacks) but scales *down* worse (a trivial dashboard needs more concepts — panes vs. widgets vs. layouts, `pn.extension`, reactive binding — than a Streamlit script). It rewards teams that will outgrow a script; it over-serves those that won't. As of 2026 the repository is actively maintained, with a large open-issue count that reflects broad surface area and heavy real-world use rather than abandonment.

## Getting Started

```bash
pip install panel        # or: conda install -c conda-forge panel
```

```python
# app.py
import panel as pn

pn.extension()                       # loads required JS/CSS; call before building UI

def model(n=5):
    return "⭐" * n

slider = pn.widgets.IntSlider(value=5, start=1, end=5)
out = pn.bind(model, n=slider)       # reactive: re-runs model when slider changes

pn.Column(slider, out).servable()    # mark as the served document
```

```bash
panel serve app.py --show --autoreload
```

The same `app.py` renders inline if the final `pn.Column(...)` is the last expression in a Jupyter cell — the notebook and server code paths are intentionally identical.

## Architecture / How It Works

Every Panel object is a **Param** `Parameterized` class. Widgets, panes (view wrappers around external objects), and layouts (`Row`, `Column`, `Tabs`, `GridStack`) all expose typed, watchable parameters. Reactivity is Param's `watch`/dependency machinery, surfaced through several APIs of increasing age:

- `pn.bind(fn, arg=widget)` — bind a function's arguments to widgets/parameters; the current recommended reactive API.
- `pn.rx(...)` — reactive expressions (added in the 1.x line) that let you write `df.rx()[col] > slider` style pipelines.
- `@pn.depends(...)` and manual `widget.param.watch(cb, 'value')` — older callback styles, still supported.

Rendering and transport are **Bokeh's**. When you serve an app, Panel builds a Bokeh document, and BokehJS in the browser maintains a WebSocket to the server. State changes on either side are diffed and synced as Bokeh model updates — this is what gives Panel true bidirectional events (clicks, hovers, selections) rather than full-page reruns. `pn.extension('plotly', 'tabulator', ...)` injects the JS/CSS bundles for the panes and widgets you intend to use; forgetting to list an extension is the most common "my plot renders blank" bug.

Custom components come in two flavors: **`ReactiveHTML`** (declare an HTML/JS template with Python-synced attributes, no build step) and full Bokeh model extensions (TypeScript, requires a compile). The `Tabulator` widget (a wrapper over the Tabulator.js grid) is the workhorse for interactive tables and is disproportionately feature-rich compared to the rest of the widget set.

Deployment targets are unusually varied: a Tornado server (Bokeh's default), or embedded under Flask/Django/FastAPI; a fully client-side app compiled to WebAssembly via `panel convert` (Pyodide/PyScript); or static export to HTML/PNG/GIF. The server path is stateful; the WASM path trades a large initial download for zero backend.

## Production Notes

**The server is stateful, per-session, and in-memory.** Each browser connection holds a live Python session with its own object graph on the server. This is what enables rich interactivity, but it means:

- A single Python process is single-threaded for callbacks (the GIL); a slow callback blocks other users on that process. Scale with `panel serve --num-procs N` (fork N processes) and/or run multiple containers behind a load balancer. WebSocket sessions require **sticky sessions / session affinity** at the load balancer — a session cannot move between processes.
- Long-lived sessions accumulate state; unreleased references (closures capturing large DataFrames, `pn.state.onload` callbacks) leak memory over hours/days. Use `pn.state.cache` for data shared across sessions and be deliberate about what each callback closes over.
- CPU-bound work should be pushed to threads (`pn.config.nthreads`), a process pool, or an external queue — not run inline in a callback.

**Bokeh version coupling is the sharpest upgrade footgun.** Panel pins a compatible Bokeh range, and the 1.0 release (2023) was gated on Bokeh 3.x[^2]. Upgrading Panel can force a Bokeh major bump; if you ship custom Bokeh model extensions or depend on other Bokeh-based libraries (HoloViews), they must move in lockstep. Pin all three together.

**Notebook vs. server discrepancies.** Rendering is *mostly* identical, but comms differ (Jupyter comms vs. server WebSocket). Bugs that appear only when served — auth, custom JS timing, static-file paths, absolute URLs — are common. Test in the actual `panel serve` target, not only the notebook.

**Authentication is built in, unusually.** `pn.config` supports OAuth providers (Auth0, Okta, GitHub, Azure, generic OIDC) and Basic auth via server flags, which is more than most Python app frameworks ship natively[^3]. It is component-level page auth, not a full authorization framework.

**WASM (`panel convert`) is real but heavy.** The Pyodide bundle plus your pure-Python dependencies can push initial loads into the tens of megabytes, and any dependency with C extensions must have a Pyodide wheel. Excellent for docs/demos and zero-infra sharing; rarely the answer for a data-heavy internal tool.

## When to Use / When Not

**Use when:**
- You need a genuine multi-page app with custom layouts, custom components, and fine-grained event handling — not just a linear script.
- You want to keep using several different plotting libraries in one app.
- You are already in the HoloViz/Param world (hvPlot, HoloViews, Datashader) and want native integration.
- You want built-in OAuth and multiple deployment shapes (server, WASM, static) from one codebase.

**Avoid when:**
- You want a five-line dashboard shipped this afternoon — Streamlit or Gradio have a shorter path to first render.
- Your team won't invest in learning Param's reactive model; the abstractions cost more than they return for simple apps.
- You need stateless horizontal scaling with no session affinity — the persistent-WebSocket model fights that.
- You only need an ML demo UI (chatbot, image in/out) — Gradio is purpose-built for that.

## Alternatives

- streamlit/streamlit — use instead when you want the fastest path from script to dashboard and can accept the full-rerun execution model.
- plotly/dash — use instead when you prefer an explicit callback graph and a React/Flask-based stateless-ish architecture.
- gradio-app/gradio — use instead for ML model demos and simple input/output interfaces.
- posit-dev/py-shiny — use instead if you want Shiny's reactive model with first-class Posit/Connect hosting.
- voila-dashboards/voila — use instead when your app is genuinely just a Jupyter notebook rendered as a page.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.5 | 2019-04 | One of the first broadly usable public releases; Param + Bokeh foundation established[^1]. |
| 0.10 | 2020-10 | Template system, growing pane/widget coverage. |
| 0.12 | 2021-08 | JupyterLab preview, notebook workflow improvements. |
| 0.14 | 2022-09 | Last of the 0.x line before the Bokeh 3 transition. |
| 1.0 | 2023-06 | Major release on Bokeh 3; reworked component model, `ReactiveHTML`, updated templates[^2]. |
| 1.3 | 2023-11 | Reactive expressions (`pn.rx`) and API consolidation in the 1.x line. |
| 1.4 | 2024 | Further reactive/API refinements and component additions. |

## References

[^1]: Panel documentation and project overview. https://panel.holoviz.org
[^2]: HoloViz blog / Panel release notes for the 1.0 release on Bokeh 3. https://panel.holoviz.org/about/releases.html
[^3]: Panel authentication how-to (OAuth / Basic auth). https://panel.holoviz.org/how_to/authentication/index.html

## Tags

python, dashboards, data-visualization, web-framework, jupyter, bokeh, holoviz, reactive, data-apps, param
