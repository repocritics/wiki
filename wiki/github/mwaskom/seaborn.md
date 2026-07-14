# mwaskom/seaborn

> A statistical plotting layer on top of matplotlib — dataset-in, styled-chart-out for tidy pandas data.

[GitHub repo](https://github.com/mwaskom/seaborn) ·
[Official website](https://seaborn.pydata.org) ·
[License: BSD-3-Clause](https://github.com/mwaskom/seaborn/blob/master/LICENSE.md)

## Overview

Seaborn is a Python visualization library that wraps matplotlib to produce
statistical graphics from tidy (long-form) data, typically pandas DataFrames.
It was written by Michael Waskom, originally to support his own neuroscience
research, and first appeared publicly around 2012[^1]. Its pitch has always
been the same: you pass a DataFrame and the names of columns, and seaborn does
the aggregation, statistical estimation, color mapping, and axis labeling that
you would otherwise hand-code against matplotlib's low-level API.

The defining tradeoff is that seaborn is a *convenience layer*, not a rendering
engine. Every seaborn figure is a matplotlib figure underneath, which means you
get matplotlib's maturity, backends, and export formats for free — but also its
constraints. When a seaborn one-liner does not produce exactly what you want,
you drop down to raw matplotlib on the returned `Axes`, and the abstraction
leaks immediately. It is excellent for exploratory analysis and publication
statistical plots; it is not an interactive or web-first tool.

Seaborn is effectively a single-maintainer project, which has kept the API
coherent and opinionated but makes release cadence slow and bus factor a real
consideration for teams standardizing on it. The library is mature and stable
rather than fast-moving; most recent energy has gone into the newer
`seaborn.objects` grammar-of-graphics interface[^2].

## Getting Started

```bash
pip install seaborn          # or: uv pip install seaborn
pip install "seaborn[stats]" # optional scipy/statsmodels extras
# conda install seaborn      # conda-forge tracks releases faster than defaults
```

```python
import seaborn as sns

sns.set_theme(style="whitegrid")          # sets global matplotlib rcParams
tips = sns.load_dataset("tips")           # downloads from mwaskom/seaborn-data

# Figure-level: seaborn owns the figure, facets by column
sns.relplot(
    data=tips, x="total_bill", y="tip",
    hue="smoker", col="time", kind="scatter",
)
```

```python
# The objects interface (v0.12+): declarative, grammar-of-graphics style
import seaborn.objects as so

(
    so.Plot(tips, x="total_bill", y="tip", color="smoker")
    .add(so.Dot())
    .add(so.Line(), so.PolyFit())
)
```

## Architecture / How It Works

Seaborn has two parallel public APIs that a new user must learn to tell apart.

**The functions API** is the historical core. It splits into two kinds of
function that look similar but behave very differently:

- **Axes-level functions** (`scatterplot`, `lineplot`, `histplot`, `kdeplot`,
  `boxplot`, `barplot`, `heatmap`, ...) draw onto a single matplotlib `Axes`.
  They accept an `ax=` argument and compose cleanly inside your own
  `plt.subplots()` layout.
- **Figure-level functions** (`relplot`, `displot`, `catplot`, `lmplot`,
  `jointplot`, `pairplot`) create and own an entire figure, usually a
  `FacetGrid`, and add faceting via `row=`/`col=`. They do **not** take `ax=`
  and cannot be dropped into an existing subplot[^3].

This axes-level vs figure-level split is the single biggest source of
confusion in seaborn, and the official docs devote a dedicated page to it[^3].

**The objects interface** (`seaborn.objects`, `so.Plot`), added in v0.12,
is a separate declarative system modeled on the grammar of graphics: you compose
`Mark`s, `Stat`s, `Scale`s, and `Move`s onto a `Plot`. It is intended as the
long-term direction but does not yet cover every capability of the older
functions[^2], so both coexist.

Statistical work (KDE bandwidth, bootstrapped confidence intervals, regression
fits) is computed in Python at plot time; scipy and statsmodels are optional
dependencies that unlock the heavier estimators. Built-in example datasets are
fetched over the network from the companion `mwaskom/seaborn-data` repo via
`load_dataset`.

## Production Notes

- **`set_theme()` mutates global state.** Seaborn styling functions change
  matplotlib's global `rcParams`. Importing seaborn and calling `set_theme()`
  restyles every subsequent matplotlib plot in the process, which surprises
  people mixing seaborn with hand-rolled matplotlib. Use `sns.axes_style()` as
  a context manager to scope it.
- **Figure-level functions fight custom layouts.** Because `relplot`/`displot`/
  `catplot` own their figure, you cannot place them into a `GridSpec` or an
  existing subplot. The standard workaround is to use the axes-level equivalent
  (`scatterplot` instead of `relplot`) with an explicit `ax=`.
- **Performance is matplotlib-bound and Python-bound.** KDE plots, large
  scatter plots, and wide `pairplot` grids get slow well before matplotlib
  itself struggles, because the statistical estimation runs in Python. For
  large N, pre-aggregate or sample before plotting.
- **Output is static.** Seaborn renders through matplotlib backends to PNG/SVG/
  PDF. There is no built-in interactivity, zoom, or web embedding; reach for a
  different library if you need that.
- **API churn across minor versions.** The old `distplot` was deprecated and
  removed in favor of `histplot`/`displot`/`ecdfplot`[^4]; several keyword and
  function renames landed across 0.9 → 0.11 → 0.12. Pin the version in
  reproducible pipelines and read the release notes before upgrading.
- **`load_dataset` needs the internet.** Tutorials lean on it, but it hits
  GitHub at call time; it will fail in air-gapped or CI-sandboxed environments.

## When to Use / When Not

**Use when:**
- Your data is already a tidy pandas DataFrame and you want statistical charts
  (distributions, regressions, categorical aggregates) with minimal code.
- You are doing exploratory data analysis or producing static figures for
  papers, reports, or notebooks.
- You want matplotlib-quality vector output without writing matplotlib.

**Avoid when:**
- You need interactive or web-native charts — use Plotly, Bokeh, or Altair.
- You are plotting very large datasets where per-point Python estimation is a
  bottleneck.
- You need fine pixel-level control over every element; at that point you are
  writing matplotlib anyway and seaborn adds an indirection.

## Alternatives

- matplotlib/matplotlib — the layer seaborn sits on; use directly when you need
  full low-level control or seaborn's abstractions get in the way.
- plotly/plotly.py — use when you need interactive, zoomable, web-embeddable
  charts rather than static images.
- altair-viz/altair — use when you want a declarative Vega-Lite grammar with
  interactivity and JSON-serializable specs.
- has2k1/plotnine — use when you specifically want R's ggplot2 grammar of
  graphics ported to Python.
- bokeh/bokeh — use when you are building interactive dashboards or streaming
  browser visualizations.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2012 | First public release; matplotlib styling + statistical helpers[^1]. |
| 0.9.0 | 2018-07 | Introduced `relplot`/`scatterplot`/`lineplot` relational API. |
| 0.11.0 | 2020-09 | New distributions API (`histplot`/`displot`); `distplot` deprecated[^4]. |
| — | 2021 | JOSS paper published, DOI 10.21105/joss.03021[^5]. |
| 0.12.0 | 2022-09 | `seaborn.objects` grammar-of-graphics interface introduced[^2]. |
| 0.13.0 | 2023-09 | Objects interface expanded; further cleanup of legacy APIs. |

## References

[^1]: Michael Waskom et al., seaborn — statistical data visualization. Repository created 2012-06. https://github.com/mwaskom/seaborn
[^2]: seaborn docs, "The seaborn.objects interface." https://seaborn.pydata.org/tutorial/objects_interface.html
[^3]: seaborn docs, "Overview of seaborn plotting functions — figure-level vs. axes-level." https://seaborn.pydata.org/tutorial/function_overview.html
[^4]: seaborn docs, distributions tutorial (replacement for the removed `distplot`). https://seaborn.pydata.org/tutorial/distributions.html
[^5]: Michael L. Waskom, "seaborn: statistical data visualization," Journal of Open Source Software, 2021. https://doi.org/10.21105/joss.03021

## Tags

python, data-visualization, statistics, matplotlib, pandas, plotting, data-science, exploratory-data-analysis, charts, grammar-of-graphics
