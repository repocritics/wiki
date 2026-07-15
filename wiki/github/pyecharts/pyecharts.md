# pyecharts/pyecharts

> A Python binding that emits Apache ECharts configuration — charts are declared in Python but rendered by JavaScript in a browser.

[GitHub repo](https://github.com/pyecharts/pyecharts) ·
[Official website](https://pyecharts.org) ·
[License: MIT](https://github.com/pyecharts/pyecharts/blob/master/LICENSE)

## Overview

pyecharts is a Python wrapper around [Apache ECharts](https://echarts.apache.org), the JavaScript visualization library originally open-sourced by Baidu[^1]. It does not draw anything itself: a pyecharts chart object accumulates an option tree that mirrors ECharts' JSON schema, and `.render()` serializes that tree and injects it into an HTML page that loads `echarts.min.js` and does the actual drawing in the browser. Understanding this — that pyecharts is a config serializer plus a template engine, not a rendering engine — explains most of its behavior and its footguns.

The project was started by chenjiandongx and first appeared in 2017[^2]. It is popular in the Chinese data community, where ECharts is the default charting library; documentation is Chinese-first, with an English translation that lags. It ships a fluent, chain-call API and covers 30+ chart types (bar, line, pie, scatter, k-line, graph, sankey, geo/map, gauge, wordcloud, 3D variants) plus 400+ bundled map files for geographic data, including native Baidu Map integration[^3].

The defining tension: because pyecharts tracks the ECharts option schema almost one-to-one, it inherits ECharts' full expressiveness but also its complexity, and it is only as current as its own release cycle. Options that a given ECharts version supports but the installed pyecharts version has not wrapped must be passed as raw dicts or `JsCode`, and stale Stack Overflow answers written against the incompatible v0.5.x API are a persistent source of confusion.

## Getting Started

```shell
pip install pyecharts -U   # v1+ requires Python 3.7+
```

```python
from pyecharts.charts import Bar
from pyecharts import options as opts

bar = (
    Bar()
    .add_xaxis(["shirt", "sweater", "tie", "pants", "coat", "heels", "socks"])
    .add_yaxis("Store A", [114, 55, 27, 101, 125, 27, 105])
    .add_yaxis("Store B", [57, 134, 137, 129, 145, 60, 49])
    .set_global_opts(title_opts=opts.TitleOpts(title="Sales", subtitle="demo"))
)
bar.render("bar.html")   # writes a self-contained HTML file
```

Rendering a static PNG requires a headless browser, not pyecharts alone:

```python
from snapshot_selenium import snapshot
from pyecharts.render import make_snapshot
make_snapshot(snapshot, bar.render(), "bar.png")  # needs selenium + a chromedriver
```

## Architecture / How It Works

Every chart class (`pyecharts.charts.*`) inherits from a base that holds an ordered dict of ECharts options. Methods like `add_xaxis`, `add_yaxis`, `set_series_opts`, and `set_global_opts` mutate that dict; they return `self`, which is what enables the chain-call style. The `pyecharts.options` (`opts`) module is a large set of dataclass-like wrappers — `TitleOpts`, `AxisOpts`, `LabelOpts`, etc. — that exist purely to give typed, discoverable Python names to ECharts' JSON keys. Each wrapper knows how to flatten itself back into the plain dict ECharts expects.

`.render()` serializes the accumulated option tree to JSON and feeds it into a Jinja2 template that produces an HTML document. That document contains a `<div>`, a `<script>` tag pulling `echarts.min.js` (and any extra assets such as map or wordcloud JS), and an inline call to `echarts.init(...).setOption(<your JSON>)`. The pixels are drawn client-side by ECharts; pyecharts' job ends at emitting text.

Consequences of this design:

- **Raw escape hatches.** Anything ECharts supports but pyecharts hasn't wrapped can still be set by passing a Python dict, or by wrapping a JavaScript function string in `JsCode(...)` — pyecharts emits it verbatim into the page. This is how formatters, custom tooltips, and event handlers work.
- **Asset resolution.** The `<script>` src is governed by `CurrentConfig.ONLINE_HOST`. By default assets are fetched from a CDN / GitHub-hosted host, so a rendered file is not truly self-contained without configuration.
- **Notebook rendering** uses a different code path: chart objects implement `_repr_html_` for Jupyter Notebook, and JupyterLab additionally requires calling `load_javascript()` to register the ECharts bundle. marimo is also supported[^4].
- **Web integration** (Flask, Django, Sanic) works by rendering just the chart fragment or the option JSON and embedding it in a server-side template or returning it to a frontend.

## Production Notes

**Offline / air-gapped deployments.** The single most common operational surprise: rendered charts pull `echarts.min.js` from a remote host by default. On a machine with no internet, or behind a strict CSP, charts silently render blank. The fix is to download the ECharts assets locally and point `CurrentConfig.ONLINE_HOST` at your own static path — this must be done deliberately.

**Static images are expensive.** There is no pure-Python path to a PNG/SVG. `make_snapshot` drives a real headless browser (selenium + chromedriver, or the deprecated snapshot-phantomjs) to screenshot the rendered page. In server/CI environments this means shipping and maintaining a browser binary, which is heavy and a frequent breakage point on version drift between Chrome and chromedriver.

**Data is embedded inline.** All series data is serialized into the HTML/JSON. Large datasets produce large files and slow client-side rendering, and there is no server-side downsampling or streaming — pyecharts is a poor fit for very large or live-updating data compared to server-push tools.

**`JsCode` is arbitrary JavaScript.** It is injected unescaped into the page. Never build `JsCode` strings from untrusted input; treat it as a script-injection surface.

**The v0.5.x → v1 break.** v1 was a full rewrite and is not API-compatible with v0.5.x[^5]. v0.5.x is unmaintained and lives on the `05x` branch. Because much of the online Q&A predates the rewrite, copy-pasted examples often fail against modern installs. v2 moved rendering onto ECharts 5.4.1+, which changed some visual defaults and option availability.

**Docs asymmetry.** Authoritative docs and the example gallery are Chinese-first; the English documentation and README trail the Chinese version in completeness.

## When to Use / When Not

**Use when:**
- Your data lives in Python and you want interactive ECharts output (zoom, tooltips, legends, animation) embedded in an HTML page or web framework.
- You need ECharts-specific chart types or the bundled map/Baidu-map geographic data.
- You are rendering to a browser or notebook and are comfortable managing JS assets.

**Avoid when:**
- You need static, publication-quality figures with no browser dependency — matplotlib is the right tool.
- You want server-side rendered pixels or streaming/live dashboards at scale.
- Your environment is offline and you cannot host the ECharts assets, or your team cannot read Chinese docs and needs mature English references.
- You want a grammar-of-graphics statistical API rather than an imperative chart builder.

## Alternatives

- plotly/plotly.py — comparable interactive-JS charting with mature English docs and a full dashboard framework (Dash); use it when English documentation and app-building matter.
- matplotlib/matplotlib — static, pure-Python rendering; use it for print/publication figures and when no browser can be involved.
- bokeh/bokeh — Python-native interactive viz with a server for streaming/large data; use it when you need server-push or live updates.
- altair-viz/altair — declarative Vega-Lite grammar of graphics; use it when you want a statistical, composable API over imperative chart assembly.
- apache/echarts — the underlying JS library; use it directly when your app is JavaScript-first and Python is not in the loop.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | First release; wraps ECharts from Python[^2]. |
| 0.5.x | pre-2019 | Supported Python 2.7 / 3.4+. Now unmaintained (`05x` branch)[^5]. |
| 1.0.0 | 2019 | Full rewrite; Python 3.6+, chain-call API, `options` module[^5]. |
| 2.0.0 | 2023 | Rendering moved to ECharts 5.4.1+. |

## References

[^1]: Apache ECharts project. https://echarts.apache.org
[^2]: pyecharts repository, created 2017-06-22. https://github.com/pyecharts/pyecharts
[^3]: pyecharts README — features list (30+ charts, 400+ map files, notebook and web-framework support). https://github.com/pyecharts/pyecharts/blob/master/README.md
[^4]: pyecharts documentation. https://pyecharts.org
[^5]: pyecharts README — v0.5.x / v1 incompatibility, ISSUE#892 and ISSUE#1033. https://github.com/pyecharts/pyecharts/issues/892

## Tags

python, data-visualization, charts, apache-echarts, plotting, javascript, interactive-charts, jupyter, web, geospatial
