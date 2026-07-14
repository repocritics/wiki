# matplotlib/matplotlib

> The foundational plotting library for Python — publication-quality static figures, and the substrate almost every other Python viz tool renders through.

[GitHub repo](https://github.com/matplotlib/matplotlib) ·
[Official website](https://matplotlib.org/stable/) ·
License: custom Matplotlib License (PSF-derived, BSD-compatible)[^1]

## Overview

Matplotlib is a 2D (and limited 3D) plotting library for Python, started by John D. Hunter in 2003 to bring MATLAB-style plotting to Python; the GitHub repository dates to 2011 and the project is now fiscally sponsored by NumFOCUS[^2]. Hunter died in 2012, and the library has since been maintained by a distributed development team. At ~23k stars and ~8.4k forks it is one of the oldest continuously maintained packages in the scientific-Python stack, and effectively the reference renderer: pandas `.plot()`, seaborn, and much of the notebook ecosystem draw through it.

Its defining tension is age. Matplotlib predates almost every modern Python API convention, and it carries two interfaces at once: a stateful, MATLAB-emulating `pyplot` module (`plt.plot()`, `plt.title()`, an implicit "current figure") and an explicit object-oriented API (`fig, ax = plt.subplots()`). Tutorials mix the two freely, which is the single biggest source of confusion for new users. The library is comprehensive and infinitely configurable, but that configurability is exposed through a large, historically-accreted surface rather than a small orthogonal one.

The other tension is interactivity. Matplotlib was designed for static, print-quality output; it renders pixels, not a scene graph you pan and zoom in a browser. For dashboards and large interactive datasets, it is usually the wrong tool — that gap is why plotly, bokeh, and altair exist.

## Getting Started

```bash
python -m pip install matplotlib
# or, in the conda ecosystem:
conda install -c conda-forge matplotlib
```

```python
import matplotlib.pyplot as plt
import numpy as np

# Prefer the explicit object-oriented API over pyplot's implicit state.
fig, ax = plt.subplots(figsize=(6, 4))

x = np.linspace(0, 2 * np.pi, 200)
ax.plot(x, np.sin(x), label="sin")
ax.plot(x, np.cos(x), label="cos")

ax.set_title("Trig functions")
ax.set_xlabel("x")
ax.legend()

fig.savefig("out.png", dpi=150)   # static file, no display needed
```

## Architecture / How It Works

Matplotlib is classically described as three layers[^3]:

1. **Backend layer** — the rendering and event abstraction: a `FigureCanvas` (the drawing surface), a `Renderer` (draws onto it), and an `Event` system. Backends split into *interactive* (Qt, GTK, Tk, wx, macosx — the topics on this repo) and *non-interactive / hardcopy* (Agg for raster PNG, plus PDF, PS, SVG). The Agg backend, built on the Anti-Grain Geometry C++ library, is the default raster renderer and does the actual pixel work for most output.
2. **Artist layer** — everything you see is an `Artist`: `Figure`, `Axes`, `Line2D`, `Text`, `Patch`, `Tick`. A `Figure` contains one or more `Axes` (a single plot region, not to be confused with "axis"); each `Axes` owns the artists drawn inside it. This is the layer the object-oriented API exposes.
3. **Scripting layer** — `pyplot`. A thin, stateful wrapper that tracks a "current figure" and "current axes" so you can issue commands without holding references. Convenient for interactive sessions; a liability in library code and multi-figure scripts.

Underneath sits the **transforms** system — an explicit pipeline mapping data coordinates → axes coordinates → figure coordinates → display pixels. It is what makes log scales, non-linear projections, and shared axes work, and also why custom coordinate work in matplotlib has a steep learning curve.

Configuration is global, through **rcParams** (`matplotlib.rcParams`, seeded from a `matplotlibrc` file and style sheets). Styles like `plt.style.use("ggplot")` are just bundled rcParams overrides. Because rcParams is process-global mutable state, changing it in one place affects every subsequent figure.

## Production Notes

- **Not thread-safe.** Matplotlib's pyplot state and figure rendering are not
  designed for concurrent use across threads. In web servers, render each
  figure in a single thread (or process), use the object-oriented API, and
  avoid the global pyplot state[^4].
- **Choose the backend before importing pyplot.** On headless servers set the
  non-interactive Agg backend explicitly (`matplotlib.use("Agg")`) *before*
  `import matplotlib.pyplot`, or set `MPLBACKEND=Agg`. Otherwise matplotlib may
  try to select a GUI backend and fail, or pull in unwanted GUI dependencies.
- **Close your figures.** `plt.figure()` / `plt.subplots()` retain figures in
  pyplot's registry; in long-running processes (Flask/Django endpoints, loops
  generating many charts) this leaks memory until you call `plt.close(fig)` or
  `plt.close("all")`.
- **Font cache cold start.** The first import after install (and inside fresh
  containers) rebuilds a font cache, which can add a multi-second delay and,
  historically, cause confusing warnings. Pre-warm it in your image build.
- **Large datasets are slow.** Matplotlib draws on the CPU with no GPU
  acceleration; scatter/line plots of millions of points are sluggish and
  produce huge vector files. Downsample, rasterize (`rasterized=True`) inside
  otherwise-vector output, or reach for datashader / pyqtgraph.
- **Two APIs, pick one.** Mixing `plt.*` stateful calls with `ax.*`
  object-oriented calls is the most common bug source. In any code that
  outlives a REPL, use `fig, ax = plt.subplots()` and never touch the implicit
  current-figure.
- **Style defaults changed in 2.0.** Matplotlib 2.0 (2017) overhauled default
  colors, fonts, and the default colormap to `viridis`[^5]; figures generated
  before and after that boundary look materially different, which matters for
  reproducing old outputs. Pin `matplotlib` and consider an explicit style.

## When to Use / When Not

**Use when:**
- You need publication-quality static figures (PNG/PDF/SVG/EPS) with precise
  control over every element.
- You are producing figures for papers, reports, or print where exact layout,
  fonts, and vector output matter.
- You want the lowest-common-denominator library that integrates with NumPy,
  pandas, seaborn, and Jupyter with zero friction.

**Avoid when:**
- You need interactive, pan/zoom, browser-native dashboards — use plotly,
  bokeh, or a JS charting library.
- You are visualizing millions of points or streaming data in real time.
- You want a concise declarative grammar of graphics rather than imperative
  drawing calls — altair or plotnine express intent in far less code.

## Alternatives

- plotly/plotly.py — use instead when you need interactive, web/browser-native
  charts and dashboards rather than static images.
- bokeh/bokeh — use instead for interactive visualization of large or streaming
  datasets served to the browser.
- mwaskom/seaborn — not a replacement but a higher-level statistical layer that
  renders *through* matplotlib; use for common statistical plots in less code.
- vega/altair — use instead when you want a concise declarative grammar of
  graphics (Vega-Lite) over imperative plotting calls.
- pyqtgraph/pyqtgraph — use instead for fast, real-time interactive plotting in
  desktop (Qt) applications.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2003 | Initial release by John D. Hunter, MATLAB-style plotting for Python[^2]. |
| 1.0 | 2010-07 | First 1.0 release; API and toolkit maturity. |
| 2.0 | 2017-01 | New default style, `viridis` default colormap, revamped defaults[^5]. |
| 3.0 | 2018-09 | Dropped Python 2 support; Python 3 only[^6]. |
| 3.8 | 2023-09 | Type hints shipped in the package; Python 3.9+ baseline. |
| 3.9 | 2024-05 | Continued API cleanup; adopted EffVer versioning scheme[^7]. |
| 3.10 | 2024-12 | Recent stable line; ongoing 3.x maintenance. |

## References

[^1]: Matplotlib License — custom PSF-derived, BSD-compatible agreement (GitHub does not map it to an SPDX identifier). https://github.com/matplotlib/matplotlib/blob/main/LICENSE/LICENSE
[^2]: Matplotlib history and NumFOCUS sponsorship. https://matplotlib.org/stable/project/history.html
[^3]: "The Lifecycle of a Plot" and architecture overview. https://matplotlib.org/stable/users/explain/quick_start.html
[^4]: Matplotlib FAQ — working with threads / embedding in web servers. https://matplotlib.org/stable/users/faq.html
[^5]: Matplotlib 2.0 release notes — style changes and default colormap. https://matplotlib.org/stable/users/prev_whats_new/dflt_style_changes.html
[^6]: Matplotlib 3.0.0 release notes — Python 2 dropped. https://matplotlib.org/stable/users/prev_whats_new/whats_new_3.0.html
[^7]: EffVer versioning scheme, adopted by matplotlib. https://jacobtomlinson.dev/effver/
[^8]: Repository metadata (stars, forks, topics, last push) via GitHub API, 2026-07. https://github.com/matplotlib/matplotlib

## Tags

python, data-visualization, plotting, matplotlib, charting, scientific-computing, data-science, publication-graphics, agg-backend, numfocus
