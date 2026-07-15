# has2k1/plotnine

> A grammar of graphics for Python — a faithful port of R's ggplot2, rendered on top of matplotlib.

[GitHub repo](https://github.com/has2k1/plotnine) ·
[Official website](https://plotnine.org) ·
[License: MIT](https://github.com/has2k1/plotnine/blob/main/LICENSE)

## Overview

plotnine implements Leland Wilkinson's *Grammar of Graphics* in Python by
closely mirroring the API of R's ggplot2[^1]. You build a plot by adding
layers with `+`: a data frame, an aesthetic mapping (`aes`), one or more
geometries (`geom_*`), statistical transforms (`stat_*`), scales, facets, and
a theme. The premise is that plots are compositions of a small set of
orthogonal primitives, so complex figures are assembled incrementally rather
than drawn imperatively. It is maintained primarily by Hassan Kibirige
(has2k1), who also authors most of its dependency stack.

The audience is people who already know ggplot2 and want the same mental model
in Python, or data analysts who prefer declarative statistical graphics over
matplotlib's imperative artist API. plotnine consumes tidy (long-format)
pandas data frames and expects columns to be mapped to visual channels — the
same "one row per observation" discipline ggplot2 demands.

The defining tension is that plotnine is a declarative grammar sitting on an
imperative, static rendering backend. Everything is compiled down to
matplotlib, which means plotnine inherits matplotlib's output quality and its
performance ceiling, produces static images rather than interactive charts,
and occasionally forces you to drop to raw matplotlib for things the grammar
does not express. It is a translation layer, not a new rendering engine.

## Getting Started

```console
$ pip install plotnine
# or: conda install -c conda-forge plotnine
```

```python
from plotnine import ggplot, aes, geom_point, stat_smooth, facet_wrap
from plotnine.data import mtcars

(
    ggplot(mtcars, aes("wt", "mpg", color="factor(gear)"))
    + geom_point()
    + stat_smooth(method="lm")
    + facet_wrap("gear")
)
```

Evaluating a `ggplot` object in a notebook renders it inline. In a script,
call `.save("out.png", dpi=300)` or `.draw()` to get the underlying
matplotlib `Figure`. `from plotnine import *` is the idiomatic (if
namespace-polluting) import the docs use.

## Architecture / How It Works

A `ggplot` object is a lazy specification, not a rendered figure. Adding
components with `__add__` accumulates layers, scales, facets, coordinates, and
theme overrides into the object. Nothing is drawn until the plot is displayed
or saved, at which point plotnine runs a build pipeline: it computes the
statistics for each layer, trains the scales on the combined data, maps data
values to aesthetic values, applies the facet layout and coordinate system,
and finally emits matplotlib artists onto a `Figure` / `Axes` grid.

The dependency graph is the important part of the story, and much of it is the
maintainer's own work:

- **matplotlib** is the rendering backend. Every geom ultimately becomes
  matplotlib primitives, and themes are largely matplotlib rcParam
  translations. plotnine's visual fidelity and its limits are matplotlib's.
- **pandas / numpy** carry the data. Input is expected to be a tidy
  DataFrame; wide data must be reshaped (melted) first.
- **mizani** provides scales, transforms, and palettes[^2] — the machinery
  behind `scale_*` and the breaks/labels formatting. It is a separate
  has2k1 package extracted from plotnine.
- **statsmodels / scipy** back the statistical geoms and stats
  (`stat_smooth`'s `lm`/`lowess`/`glm`, density estimates, etc.).
- **Formula parsing** for smoothing and similar stats has historically gone
  through patsy; newer releases moved toward formulaic.

Because the grammar is a thin, well-structured layer over matplotlib, you can
usually reach the compiled `Figure` and post-process it. But the abstraction
is one-directional: plotnine drives matplotlib, and matplotlib does not know
about the grammar, so anything you tweak imperatively afterward lives outside
plotnine's model and will not survive a re-render.

## Production Notes

**It renders static images.** There is no interactivity, no zoom/pan, no
hover tooltips, no browser widget. If a stakeholder wants an interactive
dashboard, plotnine is the wrong tool — this is a fundamental design choice,
not a missing feature.

**Performance is matplotlib's performance.** Large scatter plots (hundreds of
thousands of points), many-panel facets, and heavy statistical layers get slow
because each becomes matplotlib artists. There is no WebGL/datashader fast
path built in; for big data you pre-aggregate or bin before plotting.

**API stability across the 0.x line.** plotnine spent its whole life below
1.0, and minor releases have carried real breaking changes — deprecations of
argument names, theme element renames, and shifts in default behavior between
0.x versions. Pin the version in reproducible pipelines and read release notes
before upgrading; code that worked on an older minor is not guaranteed to run
unchanged.

**Dependency coupling to matplotlib and mizani.** A matplotlib or mizani
upgrade can change rendering or break plotnine before plotnine itself
releases a compatible version. Because the same maintainer owns plotnine and
mizani, those tend to move together, but matplotlib is external and
occasionally forces reactive fixes.

**Argument order and tidy-data expectations bite newcomers.** `ggplot(data,
mapping)` order, `aes()` referencing column names as strings, the long-format
data requirement, and matplotlib-level font configuration (missing glyphs
render as tofu boxes) are the most common early stumbling blocks for people
coming from matplotlib rather than R.

## When to Use / When Not

**Use when:**
- You know ggplot2 and want the same grammar in a Python/pandas workflow.
- You produce static, publication- or report-quality figures.
- Your plots are statistical and layered, and you value declarative
  composition over imperative drawing.
- You already have tidy DataFrames and want faceting and scales for free.

**Avoid when:**
- You need interactivity, streaming, or web-embedded charts.
- You are plotting very large datasets without pre-aggregation.
- You want fine-grained imperative control of every artist — use matplotlib
  directly.
- You dislike the tidy-data constraint or don't want a ggplot2 mental model.

## Alternatives

- mwaskom/seaborn — statistical plotting on matplotlib with a lighter,
  less strict API; use instead when you want good defaults without committing
  to the full grammar.
- vega/altair — declarative grammar (Vega-Lite) that renders interactive
  browser charts; use when you need interactivity and a JSON-serializable spec.
- plotly/plotly.py — interactive, web-first charts and dashboards; use when
  the deliverable is an interactive figure, not a static image.
- JetBrains/lets-plot — another ggplot2-style grammar for Python with its
  own (non-matplotlib) backend and interactivity; use when you want the
  grammar plus interactive output.
- matplotlib/matplotlib — the layer plotnine sits on; use directly when you
  need imperative control the grammar can't express.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2017 | Initial public release; maintained grammar-of-graphics implementation for Python, ggplot2-style API[^1]. |
| 0.8.0 | 2021 | Modernized dependency requirements during the 0.x line. |
| 0.12.x | 2023 | Continued matplotlib/pandas compatibility work. |
| 0.13.x | 2024 | Ongoing API and scales evolution (mizani-backed). |
| 0.14.x | 2024–2025 | Recent minor line; still pre-1.0. |

## References

[^1]: plotnine README and documentation — "an implementation of a grammar of graphics in Python based on ggplot2." https://plotnine.org
[^2]: mizani — scales, transforms and palettes used by plotnine. https://github.com/has2k1/mizani

## Tags

python, data-visualization, grammar-of-graphics, ggplot2, matplotlib, plotting, pandas, statistical-graphics, dataframe, charting
