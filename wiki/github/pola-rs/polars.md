# pola-rs/polars

> A columnar DataFrame query engine written in Rust, with a lazy optimizer and a Python front end positioned as a pandas replacement.

[GitHub repo](https://github.com/pola-rs/polars) ·
[Official website](https://docs.pola.rs) ·
[License: MIT](https://github.com/pola-rs/polars/blob/main/LICENSE)

## Overview

Polars is an analytical query engine for tabular data, written in Rust and built on the Apache Arrow columnar memory layout[^1]. It started around 2020 as Ritchie Vink's Rust DataFrame library and grew into a multi-language project with a dominant Python binding, plus Rust, Node.js, and R front ends. The 1.0 release landed in July 2024[^2], after several years of frequent pre-1.0 breaking changes. As of 2026 it is ~39k stars and pushed daily — one of the most active data-tooling projects on GitHub, with a correspondingly large open-issue count (~2,800) that reflects surface area and active triage rather than abandonment.

The defining design choice is the split between an **eager** API (execute immediately, like pandas) and a **lazy** API (`LazyFrame`) that builds a query plan and optimizes it before execution — predicate pushdown, projection pushdown, slice pushdown, common-subplan elimination, and more[^3]. The second defining choice is the **expression API**: operations like `pl.col("x").sum().over("group")` are composable expression objects evaluated inside contexts (`select`, `with_columns`, `filter`, `group_by().agg()`), which lets the engine parallelize and vectorize across columns automatically.

The central tension: Polars is genuinely faster and more memory-efficient than pandas for most analytical workloads, but it deliberately breaks with pandas conventions — no row index, strict null/NaN distinction, and an expression grammar you have to learn. It is a replacement, not a drop-in, and the ecosystem around pandas (plotting, ML libraries, tutorials) is still far larger.

## Getting Started

```sh
pip install polars          # Python
# cargo add polars --features lazy   # Rust
```

```python
import polars as pl

# Lazy pipeline — nothing runs until .collect()
result = (
    pl.scan_csv("events.csv")                 # lazy CSV scan
    .filter(pl.col("value") > 0)
    .group_by("category")
    .agg(pl.col("value").sum().alias("total"))
    .sort("total", descending=True)
    .collect()                                # optimize + execute
)
print(result)
```

`scan_csv`/`scan_parquet` keep the scan lazy so pushdowns can skip columns and row groups; `read_csv` materializes eagerly.

## Architecture / How It Works

Polars is layered as a set of Rust crates under one workspace, with `py-polars` (a PyO3 extension) wrapping the core for Python[^4]:

- **Memory** — data lives in Arrow-compatible columnar arrays. Polars ships its own Arrow implementation (originally derived from `arrow2`, now the internal `polars-arrow`) rather than depending on the official `arrow-rs`, which is a recurring source of confusion when interoperating with other Arrow tooling.
- **Series / ChunkedArray** — a column is a `ChunkedArray` of one or more Arrow chunks; concatenation appends chunks rather than copying, so `rechunk()` exists to consolidate when a contiguous layout matters.
- **Expressions** — the user-facing grammar. An expression is a lazy description of a column transformation; the same expression runs in eager or lazy mode.
- **LazyFrame + optimizer** — building a query produces a logical plan. The optimizer rewrites it (pushdowns, projection pruning, subplan reuse) into a physical plan before execution.
- **Execution engines** — the default in-memory engine is multi-threaded over Rayon. A separate **streaming engine** processes larger-than-RAM data in morsels; it was rewritten and is invoked with `collect(engine="streaming")`. Not every operation is supported in streaming mode.

Parallelism is automatic and per-operation: Polars fans work across CPU cores without the caller managing threads, and uses SIMD where the data type allows. The cost model assumes work stays inside Rust — the moment you call a Python UDF, the engine drops to single-threaded, row-by-row Python.

## Production Notes

**Python UDFs are the primary footgun.** `map_elements` (per-row Python) breaks out of the vectorized engine and is often 10–100× slower than an equivalent native expression. The rule is: express everything in the expression API; reach for a UDF only when no native operation exists, and prefer `map_batches` (whole-Series) over `map_elements` (per-element).

**Lazy is where optimization happens.** Eager `read_*` + method chaining runs each step immediately with no pushdown. For non-trivial pipelines, use `scan_*` + `collect()` so the optimizer can prune columns and skip Parquet row groups. `explain()` prints the physical plan and is the first debugging tool.

**Streaming is partial.** `engine="streaming"` handles many but not all operations; unsupported nodes force the whole query back into memory, and it is easy to assume a query streams when it silently materializes. Verify with `explain(streaming=True)` and test against realistic data sizes.

**null vs NaN.** Polars keeps `null` (missing) and `NaN` (float not-a-number) as distinct values, unlike pandas which historically conflated them. Code ported from pandas that assumes `isna()` catches both needs auditing — use `is_null()` and `is_nan()` deliberately.

**Categorical string cache.** Comparing or joining `Categorical` columns from different sources requires a shared string cache (`pl.StringCache()` or the global toggle); mismatched physical encodings otherwise raise or silently misbehave. This is global state and interacts badly with concurrency.

**Version churn.** Pre-1.0 releases broke APIs often (e.g., `groupby` → `group_by`, `apply` → `map_elements`/`map_batches`). 1.0 committed to semantic versioning[^2], but the project still ships fast; pin versions in production and read release notes before minor upgrades. Arrow interop with external tools can also break across versions because Polars manages its own Arrow layer.

**No index.** There is no pandas-style row index. Row selection is positional or predicate-based. This removes a large class of pandas alignment bugs but means index-centric pandas code must be rewritten, not translated.

## When to Use / When Not

**Use when:**
- You have analytical workloads (aggregations, joins, group-bys) that are slow or memory-bound in pandas.
- Your data is larger than RAM but fits a streaming or out-of-core pass.
- You want a single expression grammar that runs identically eager and lazy, with automatic multi-threading.
- You are building in Rust and want a native DataFrame layer without Python.

**Avoid when:**
- You depend heavily on the pandas ecosystem (scikit-learn pipelines, seaborn, domain libraries that take/return `DataFrame`).
- Your logic is inherently row-wise and can't be expressed vectorially, so it would live in Python UDFs anyway.
- You need SQL as the primary interface across many storage backends — a SQL-first engine fits better.
- You need distributed compute across a cluster today (single-node is the core design; distributed is a separate managed offering[^5]).

## Alternatives

- pandas-dev/pandas — use pandas when ecosystem breadth, the row index, or team familiarity outweigh raw speed.
- duckdb/duckdb — use DuckDB when SQL is your primary interface and you want an embedded OLAP engine over files.
- apache/arrow — use Arrow directly when you need the low-level columnar format and interchange, not a DataFrame API.
- rapidsai/cudf — use cuDF when you have NVIDIA GPUs and want GPU-accelerated DataFrames (Polars also exposes a GPU engine built on it).
- dask/dask — use Dask when you need to distribute pandas-like workloads across a cluster.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-05 | Repo created; Rust DataFrame library by Ritchie Vink[^1]. |
| 0.x | 2021–2023 | Rapid growth of the Python binding; frequent breaking API changes. |
| — | 2023 | Company formed to build managed/cloud offerings around the OSS core[^5]. |
| 1.0.0 | 2024-07 | First stable release; semantic-versioning commitment[^2]. |
| 1.x | 2024–2026 | New streaming engine, GPU engine (cuDF) integration, ongoing 1.x line. |

## References

[^1]: Polars user guide — "Introduction". https://docs.pola.rs/
[^2]: Polars blog — "Polars 1.0 release" (2024-07). https://pola.rs/posts/polars-1.0.0/
[^3]: Polars user guide — "Lazy API" and query optimizations. https://docs.pola.rs/user-guide/lazy/optimizations/
[^4]: Polars repository README and `pyo3-polars` bindings. https://github.com/pola-rs/polars/tree/main/pyo3-polars
[^5]: Polars managed/distributed offering. https://cloud.pola.rs/

## Tags

rust, python, dataframe, query-engine, apache-arrow, columnar, lazy-evaluation, out-of-core, data-analytics, pandas-alternative, olap
