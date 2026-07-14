# dask/dask

> Parallel computing for Python that mirrors the NumPy, pandas, and list APIs on top of a task-graph scheduler.

[GitHub repo](https://github.com/dask/dask) ·
[Official website](https://dask.org) ·
[License: BSD-3-Clause](https://github.com/dask/dask/blob/main/LICENSE.txt)

## Overview

Dask is a parallel-computing library for Python, started in 2015 by Matthew Rocklin at Continuum Analytics (now Anaconda) and developed under NumFOCUS[^1]. Its pitch is continuity: `dask.array` looks like NumPy, `dask.dataframe` looks like pandas, `dask.bag` looks like a parallel list, and `dask.delayed` / `dask.futures` let you parallelize arbitrary Python. The same code that runs on a laptop thread pool scales, largely unchanged, to a thousand-core cluster.

The defining tension is that "looks like pandas/NumPy" is a partial promise, not a drop-in one. Dask reimplements a large subset of those APIs by building a task graph of many small NumPy/pandas operations and executing it lazily; anything outside that subset (some indexing patterns, certain pandas methods, row-wise ordering assumptions) either is unimplemented or forces an expensive shuffle. Getting good performance requires understanding partition sizing, graph overhead, and the memory behavior of the scheduler — knowledge that the familiar surface API actively hides. Dask lowers the cost of *starting* parallel work and raises the cost of *not understanding* it.

Dask sits in the PyData stack rather than competing with it: it orchestrates NumPy and pandas rather than replacing them, which is both its strength (no new data model to learn) and its ceiling (it inherits their single-partition performance and their sharp edges).

## Getting Started

```bash
pip install "dask[complete]"          # arrays, dataframes, distributed, diagnostics
# or the local-only core:
pip install dask
```

```python
import dask.dataframe as dd

# Lazily read many parquet files as one partitioned dataframe
df = dd.read_parquet("s3://bucket/events/*.parquet")

# Familiar pandas-style API — but nothing has executed yet
result = (
    df[df.amount > 0]
      .groupby("customer_id")
      .amount.sum()
)

# .compute() materializes the task graph into a single pandas Series
print(result.compute())
```

```python
import dask

@dask.delayed
def load(path): ...
@dask.delayed
def process(x): ...

# Build a graph of arbitrary Python, then run it in parallel
tasks = [process(load(p)) for p in paths]
dask.compute(*tasks)
```

## Architecture / How It Works

Dask separates into two layers that are usually discussed together but ship in different repositories.

**Collections + graphs (this repo).** High-level collections (`array`, `dataframe`, `bag`) and the low-level `delayed` / `futures` interfaces all do one thing: build a *task graph* — a directed acyclic graph whose nodes are plain Python callables and whose edges are data dependencies. A `dask.dataframe` is internally a sequence of pandas DataFrames (partitions) split along the index; a `dask.array` is a grid of NumPy chunks. Operations are lazy and compose graphs; nothing runs until `.compute()` (or `.persist()`).

**Schedulers.** A scheduler walks the graph and executes it. Dask ships several:
- **Synchronous** — single-threaded, mainly for debugging.
- **Threaded** — default for arrays/dataframes; good when work is in NumPy/pandas C code that releases the GIL.
- **Multiprocessing** — default fallback for `bag`; avoids the GIL but pays serialization cost.
- **Distributed** — lives in the separate `dask/distributed` package: a central scheduler, many workers, and a client. It is the production path and adds a dashboard, work stealing, data locality, and spill-to-disk. Most non-trivial Dask deployments use it even on a single machine.

Since 2024 `dask.dataframe` is built on a new expression / query-optimization layer (formerly the `dask-expr` project, now folded in) that reorders and prunes operations — column projection, filter pushdown, and smarter shuffles — before graph materialization[^2]. This changed the internals substantially while keeping the surface API stable, and is now the default.

The coupling story: the collections are only as good as the scheduler executing them, and the distributed scheduler's memory model (each worker holds a slice of data in RAM, spilling to disk under pressure) is where most real-world behavior — good and bad — actually comes from.

## Production Notes

**Partition sizing is the whole game.** The most common performance and out-of-memory problems trace back to partitions that are too large (a single partition must fit comfortably in one worker's RAM) or too many tiny partitions (per-task scheduler overhead dominates). A rough target of ~100 MB per partition is the usual starting heuristic; `repartition()` is a routine tuning step, not an edge case.

**Shuffles are expensive.** Any operation that reorganizes data across partitions — `set_index`, `merge`/`join` on a non-index column, `groupby().apply` — triggers a shuffle that moves data between all workers. These dominate runtime and memory. Structuring pipelines to shuffle once (and reusing the resulting index) is the main optimization lever.

**Debugging is graph-shaped, not stack-shaped.** Because execution is lazy and distributed, a Python traceback points at graph construction, not the failing task. The distributed dashboard (task stream, worker memory, progress) is effectively required for diagnosing real workloads; treating it as optional is a common mistake.

**Memory and the "unmanaged memory" trap.** Distributed workers report managed (Dask-tracked) memory separately from unmanaged (leaked, fragmented, or held by libraries). Long-running workers accumulate unmanaged memory and get killed by the nanny; the fixes (smaller partitions, `client.restart()`, tuning spill thresholds) are operational knowledge that the API does not surface.

**Version pinning across the split.** `dask` and `distributed` are released together and must be kept at matching versions; mixing them causes protocol errors. Calendar versioning (YYYY.MM.X) makes upgrades frequent, and behavior can shift between releases without a major-version signal.

**It is not always the right tool for the size.** For data that fits in RAM, plain pandas or Polars is faster and simpler than Dask — the graph and scheduler overhead only pays off at true out-of-core or multi-machine scale.

## When to Use / When Not

**Use when:**
- Your data does not fit in memory but your logic is already NumPy/pandas-shaped and you want to keep it.
- You need to scale existing Python code to a cluster without adopting a new data model or JVM.
- You want incremental adoption — parallelize one bottleneck with `delayed`/`futures` rather than rewriting.
- You are already in the PyData/SciPy ecosystem (xarray, scikit-learn, RAPIDS all integrate with Dask).

**Avoid when:**
- The data fits in RAM: pandas or Polars will be faster with less machinery.
- You need heavy relational shuffles/joins at scale: a purpose-built engine (Spark, or a warehouse) is often more predictable.
- You want single-node maximum throughput on modern hardware: Polars' vectorized engine typically wins.
- Your team cannot invest in understanding partitioning and the distributed dashboard — the familiar API will mislead you.

## Alternatives

- ray-project/ray — general distributed-Python framework; lower-level and broader (ML serving, RL), use instead of Dask when you need task/actor parallelism beyond dataframe/array workloads.
- apache/spark — mature JVM-based cluster engine; use instead when you need battle-tested large-scale SQL/joins and can accept the JVM and PySpark boundary.
- pola-rs/polars — single-node (with streaming/out-of-core) vectorized dataframe engine; use instead when data fits one machine and raw speed matters more than clustering.
- modin-project/modin — drop-in pandas replacement that can use Dask or Ray as a backend; use instead when you want stricter pandas parity on one node.
- rapidsai/cudf — GPU dataframes; use instead (or with Dask via dask-cudf) when you have NVIDIA GPUs and want to accelerate columnar ops.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Initial release: `array`/`bag`/`dataframe` collections + local schedulers[^1]. |
| distributed | 2016 | `dask.distributed` scheduler released as a separate package. |
| 1.0 | 2018-11 | API stabilization milestone. |
| 2.0 | 2019-06 | Major cleanup; distributed scheduler maturity. |
| CalVer | ~2020 | Switched to calendar versioning (YYYY.MM.X); frequent lockstep `dask`/`distributed` releases. |
| dask-expr default | 2024 | New query-optimization/expression layer becomes the default `dask.dataframe` backend[^2]. |

## References

[^1]: Dask documentation and project history. https://docs.dask.org/en/stable/ · https://www.dask.org
[^2]: Dask blog, "Dask DataFrame is Fast Now" / dask-expr query optimizer becoming the default dataframe backend (2024). https://docs.dask.org/en/stable/dataframe.html

## Tags

python, parallel-computing, distributed-computing, dataframe, numpy, pandas, task-scheduling, big-data, pydata, out-of-core, scientific-computing
