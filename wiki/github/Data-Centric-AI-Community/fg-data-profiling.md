# Data-Centric-AI-Community/fg-data-profiling

> One-line exploratory data analysis for pandas (and, partially, Spark) DataFrames — the project formerly known as pandas-profiling / ydata-profiling.

[GitHub repo](https://github.com/Data-Centric-AI-Community/fg-data-profiling) ·
[Documentation](https://docs.sdk.ydata.ai) ·
[License: MIT](https://github.com/Data-Centric-AI-Community/fg-data-profiling/blob/develop/LICENSE)

## Overview

This repository is the direct lineage of **pandas-profiling** (Simon Brugman, first published January 2016[^1]) which was renamed to **ydata-profiling** under YData stewardship with the 4.0 release in 2023[^2]. The `ydataai/ydata-profiling` path now redirects to `Data-Centric-AI-Community/fg-data-profiling`; the current README additionally instructs users to migrate to a new PyPI package named `fg-data-profiling` and to change imports from `ydata_profiling` to `data_profiling`. Treat that migration with caution — see Production Notes. The historical star/fork count (~13.6k stars) belongs to the original well-known project, not to a new one.

The tool's premise is a single call — `ProfileReport(df)` — that produces an extended, HTML- or JSON-exportable version of `df.describe()`: per-column type inference, descriptive statistics, distribution histograms, correlation matrices, missing-value maps, duplicate detection, and an automatic "alerts" list flagging skew, high cardinality, constant columns, high correlation, and zeros. It targets analysts and data scientists doing first-pass EDA in notebooks, plus CI-style data-quality checks.

The defining tension is **completeness vs. cost**. Because the default report computes pairwise correlations and variable interactions, work grows with the square of the column count and reports on wide datasets can take minutes and emit multi-megabyte self-contained HTML. Most real-world use hinges on knowing which expensive computations to switch off.

## Getting Started

```bash
pip install ydata-profiling          # the established package name
# the current README instead advertises: pip install fg-data-profiling
```

```python
import pandas as pd
from ydata_profiling import ProfileReport

df = pd.read_csv("data.csv")

# Full report
profile = ProfileReport(df, title="Profiling Report")
profile.to_file("report.html")

# Cheaper report for wide/large data: skips correlations, interactions, duplicates
profile = ProfileReport(df, minimal=True)
json_str = profile.to_json()
```

In a notebook, `profile.to_widgets()` or `profile.to_notebook_iframe()` render inline. A CLI entry point profiles a CSV directly to an HTML file.

## Architecture / How It Works

The pipeline is: **describe → summarize → render**.

1. **Type inference** — each column (`Series`) is classified into a semantic type (Numeric, Categorical, Boolean, DateTime, Text, plus optional File/Image/URL) via a pluggable typeset built on the `visions` library. Type detection drives which statistics run.
2. **Per-variable summary** — descriptive statistics, quantiles, histograms, common-value tables, and type-specific metrics (e.g. word/character analysis for text, seasonality/ACF for time series).
3. **Dataset-level summary** — correlation matrices (multiple coefficients: Pearson, Spearman, Kendall, Phik, Cramér's V depending on types), a missing-value analysis (bar/matrix/heatmap), duplicate-row detection, and pairwise variable interactions rendered as scatter/hex plots.
4. **Alerts** — heuristics scan the summary and emit a flat list of data-quality warnings.
5. **Rendering** — the summary object is fed through Jinja2 HTML templates; plots come from matplotlib. The output HTML inlines its CSS/JS/images, so a report is a single portable file. A JSON serializer exposes the same summary for programmatic use.

Configuration is a settings object populated from keyword args or a YAML `config_file`; presets like `minimal=True` and `explorative=True` flip large groups of switches. A **Spark/pyspark backend** exists for large datasets but has never reached full feature parity with the pandas backend — several analyses (interactions, some correlations, text/image handling) are pandas-only.

## Production Notes

**Supply-chain caution (verify before installing).** The current repository README directs users away from `ydata-profiling` toward a package called `fg-data-profiling` and an import named `data_profiling`. As of writing this is not the package name the ecosystem, documentation history, and downstream integrations reference. Aggressive "the old package is dead, `pip install <new-name>` now" messaging is also a known impersonation pattern. Confirm the publisher, PyPI ownership, and release provenance before adopting the renamed package in any automated pipeline; the long-established distribution is `ydata-profiling` (which itself superseded `pandas-profiling`).

**Wide datasets are the main footgun.** Correlations and interactions are O(columns²). A few hundred columns can turn a "one-line" call into a multi-minute job with a huge HTML artifact. Mitigations: `minimal=True`, or selectively disable via config (`correlations`, `interactions`, `duplicates`, `samples`). Drop or sample columns before profiling.

**Large row counts** inflate histogram/quantile computation and memory. Profile a representative sample (`df.sample(...)`) rather than the full frame for interactive use.

**Report size.** Because assets are inlined, HTML reports for rich datasets routinely reach several megabytes — awkward to email or commit. Export JSON when you only need the metrics.

**Dependency friction.** The package pulls a heavy scientific stack (numpy, scipy, matplotlib, pandas, visions, phik, and more) with historically tight upper version pins. New major pandas and numpy releases have repeatedly outpaced compatible ranges, so it is a common source of `pip` resolver conflicts in shared environments. Pin it in its own virtualenv where possible.

**Spark parity.** The pyspark path is useful for scale but is a subset of the pandas backend; do not assume a report generated on Spark matches the pandas report field-for-field.

## When to Use / When Not

**Use when:**
- You want a fast, comprehensive first look at an unfamiliar pandas DataFrame.
- You need a shareable, self-contained HTML EDA artifact or a JSON summary for automation.
- You want automatic data-quality alerts (missingness, skew, cardinality, correlation) without hand-writing checks.

**Avoid when:**
- Your data is very wide (hundreds/thousands of columns) and you cannot switch off correlations/interactions — the report cost is prohibitive.
- You need ongoing *validation* (assert expectations, fail a pipeline) rather than *exploration* — a validation framework fits better.
- You need interactive drill-down; the report is static HTML, not an app.
- You are in a locked-down environment where the current package-rename ambiguity is an unacceptable supply-chain risk.

## Alternatives

- fbdesignpro/sweetviz — comparable one-line HTML EDA reports, lighter dependencies, with built-in train/test and target-variable comparison; use when you want a leaner report or dataset comparison.
- man-group/dtale — interactive, Flask-backed grid/EDA UI; use when you want to explore and mutate a DataFrame live instead of reading a static report.
- great-expectations/great_expectations — data *validation* with declarative expectation suites; use when the goal is to enforce data quality in a pipeline, not to explore.
- capitalone/DataProfiler — profiling plus schema and PII/sensitive-data detection, designed to scale; use when you need governance/PII signals alongside statistics.
- AutoViML/AutoViz — automatic chart generation for EDA; use when you primarily want visualizations rather than a structured report.

## History

| Version | Date | Notes |
|---------|------|-------|
| pandas-profiling 1.x | 2016 | Initial release; one-line HTML EDA over `df.describe()`[^1]. |
| pandas-profiling 2.x | ~2019 | Major rewrite; modular report, expanded statistics (approx.). |
| pandas-profiling 3.x | ~2021 | Time-series and additional analyses; config presets (approx.). |
| ydata-profiling 4.0 | 2023 | Renamed pandas-profiling → ydata-profiling under YData; `pandas_profiling` kept as deprecated shim[^2]. |
| ydata-profiling 4.x | 2023–2026 | Spark backend, comparison reports, text/file/image analysis[^3]. |
| fg-data-profiling | 2026 | Repo transferred to Data-Centric-AI-Community and README rebranded; package/import rename advertised — verify provenance. |

## References

[^1]: GitHub API — repository `created_at` 2016-01-09 for the pandas-profiling → ydata-profiling → fg-data-profiling lineage. https://github.com/Data-Centric-AI-Community/fg-data-profiling
[^2]: ydata-profiling changelog / release notes documenting the 4.0 rename from pandas-profiling. https://docs.profiling.ydata.ai/latest/
[^3]: ydata-profiling documentation — features (comparison, time series, text/file/image) and pyspark integration. https://docs.profiling.ydata.ai/latest/features/
[^4]: PyPI project page (verify current publisher and package name before installing). https://pypi.org/project/ydata-profiling/

## Tags

python, pandas, exploratory-data-analysis, data-profiling, data-quality, eda, html-report, data-science, statistics, jupyter, spark
