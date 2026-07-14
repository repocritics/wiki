# rapidsai/cudf

> GPU-accelerated DataFrame library with a pandas-compatible API, built on Apache Arrow columnar memory and CUDA.

[GitHub repo](https://github.com/rapidsai/cudf) ·
[Official docs](https://docs.rapids.ai/api/cudf/stable/) ·
[License: Apache-2.0](https://github.com/rapidsai/cudf/blob/main/LICENSE)

## Overview

cuDF (pronounced "KOO-dee-eff") is the DataFrame library of NVIDIA's RAPIDS suite. It executes tabular operations — filters, joins, groupbys, string ops, parquet/CSV/ORC I/O — on NVIDIA GPUs while presenting an API that deliberately mirrors pandas. The stated goal is that existing pandas-shaped analytics run faster on a GPU without a rewrite. It is maintained primarily by NVIDIA engineers and has been public since 2017[^1].

The project is not a single library but a stack. `libcudf` is a CUDA C++ library holding the Arrow-compliant column types and the actual kernels; `pylibcudf` exposes it through Cython; `cudf` is the pandas-like Python surface; `cudf.pandas` is a zero-code-change shim that intercepts `import pandas` and dispatches to the GPU with CPU fallback; `cudf-polars` is a GPU execution engine for Polars' lazy API; and `dask-cudf` distributes cuDF frames across multiple GPUs[^2]. Each layer targets a different willingness to change existing code.

The defining tension is the memory boundary. cuDF is fast because data lives in GPU VRAM and never crosses the PCIe bus mid-computation — but VRAM is small (tens of GB) relative to host RAM, and any operation cuDF has not implemented forces a fallback or an error. Getting value from cuDF is largely a question of keeping the working set on-device and staying inside the supported API subset. It is an accelerator for a class of workloads, not a drop-in that speeds up arbitrary pandas code uniformly.

## Getting Started

Requires a supported NVIDIA GPU, a matching CUDA driver, and Linux (Windows via WSL2). Install the wheel whose `-cu##` suffix matches your CUDA major version[^3]:

```bash
# CUDA 12
pip install cudf-cu12

# or via conda from the rapidsai channel
conda install -c rapidsai -c conda-forge cudf
```

Native cuDF API, close to pandas:

```python
import cudf

df = cudf.read_parquet("data.parquet")
result = df.dropna().groupby(["A", "B"]).mean()
```

Or accelerate unmodified pandas code with the `cudf.pandas` shim — GPU where supported, transparent CPU fallback otherwise:

```bash
python -m cudf.pandas script.py   # script.py contains plain `import pandas as pd`
```

```python
# in a Jupyter notebook, load the extension before importing pandas
%load_ext cudf.pandas
import pandas as pd
```

## Architecture / How It Works

At the core is **libcudf**, a CUDA C++ library. A cuDF column is a contiguous device buffer plus an optional null bitmask, following the Apache Arrow columnar layout — which is what makes zero-copy interchange with Arrow, Spark, and the wider ecosystem possible. Algorithms (hash joins, sort/hash groupbys, string ops) are hand-written CUDA kernels, many built on the Thrust/CUB primitives.

Device memory is managed through **RMM** (RAPIDS Memory Manager), a separate library. Because `cudaMalloc` is slow and synchronizing, RMM front-loads a large pool allocator; most production cuDF setups configure an RMM pool at startup, and allocation strategy (pool, managed/unified, async) is a real tuning knob rather than an implementation detail.

The Python layers are thin. `pylibcudf` is Cython over libcudf; the `cudf` package adds pandas-compatible semantics (index alignment, dtype coercion, `.loc`/`.iloc`) on top. **`cudf.pandas`** works by proxying pandas objects: it wraps types so operations attempt the GPU path and, on any unsupported operation or dtype, silently copy back to host and run real pandas. That fallback is what makes "zero code change" honest, but it also means a single unsupported call inside a hot loop can quietly reintroduce host round-trips.

**`cudf-polars`** is architecturally different: rather than reimplementing an API, it translates a Polars logical plan into libcudf operations, acting as a GPU engine behind `collect(engine="gpu")`[^2]. **`dask-cudf`** partitions a logical DataFrame into many cuDF frames and schedules them across GPUs, which is the sanctioned path for datasets larger than one GPU's memory.

## Production Notes

**Data must fit in VRAM.** The single largest operational constraint. A DataFrame plus the intermediate buffers a join or groupby allocates must fit in GPU memory, and peak usage during an operation can be several times the input size. Mitigations, roughly in order: RMM managed/unified memory (oversubscribe into host RAM at a performance cost), spilling, and moving to `dask-cudf` for out-of-core / multi-GPU. Treat "will this fit" as a design question, not a runtime surprise.

**`cudf.pandas` fallback is invisible.** The shim never errors on unsupported operations — it falls back to CPU pandas, including the device-to-host copy. Code can appear to "work on GPU" while spending most of its time on host round-trips. Profile with the provided `cudf.pandas` profiler to see which calls actually ran on the GPU before assuming a speedup.

**Semantic differences from pandas.** cuDF is not bit-identical to pandas. Notable divergences include: row-order after a groupby is not guaranteed unless you sort; null handling and some dtype-promotion rules differ; certain object/`apply` paths are unsupported or slow; and floating-point reductions can differ in the last bits because GPU reduction order differs. Test numerical outputs, don't assume equality.

**CUDA / driver / package version coupling.** The wheel suffix (`-cu12`, `-cu13`) must match the installed CUDA runtime, which must be compatible with the driver. RAPIDS ships on a **CalVer** cadence (`YY.MM`, roughly every two months), and cuDF versions are pinned to align across the whole RAPIDS stack (cuml, cugraph, rmm, dask-cudf). Upgrading one RAPIDS library generally means upgrading all of them to the same release; mixing versions is unsupported.

**Startup cost and small data.** GPU kernels have launch overhead and the first allocation initializes the RMM pool. On small inputs cuDF can be slower than pandas end-to-end. The GPU wins on wide/tall data and I/O-heavy parquet workloads, not on kilobyte frames in a tight loop.

## When to Use / When Not

**Use when:**
- Your working set fits (or can be partitioned to fit) in GPU memory and you run heavy groupbys, joins, or parquet/CSV/ORC I/O.
- You have existing pandas code and want to try acceleration via `cudf.pandas` with minimal changes.
- You already run on NVIDIA GPUs (Spark RAPIDS, deep-learning preprocessing, ETL feeding a GPU model).
- You use Polars and want a GPU execution engine behind the same lazy plans.

**Avoid when:**
- You have no NVIDIA GPU, or must support heterogeneous/CPU-only deployment — there is no CPU-only build.
- Data is small enough that pandas or Polars on CPU is already fast; GPU overhead dominates.
- You depend on the long tail of pandas behavior (obscure dtypes, `apply` with arbitrary Python, exact row ordering) — fallback erodes the benefit.
- You need a stable, slowly-moving dependency; the CalVer stack upgrades in lockstep every couple of months.

## Alternatives

- pandas-dev/pandas — the CPU baseline and API cuDF imitates; use it when data fits in RAM or no GPU is available.
- pola-rs/polars — fast multithreaded CPU DataFrame (Rust); use for large CPU workloads, and note cudf-polars is its GPU backend rather than a competitor.
- duckdb/duckdb — in-process OLAP SQL engine with larger-than-memory execution on CPU; use when the workload is SQL-shaped and GPU isn't warranted.
- dask/dask — scaling framework; use dask-cudf when data exceeds one GPU or you need multi-GPU/multi-node.
- apache/arrow — the columnar memory and compute layer underneath; use directly when you need format interchange rather than a DataFrame API.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | Repository created; early GPU DataFrame prototype[^1]. |
| RAPIDS launch | 2018-10 | cuDF folded into the RAPIDS suite, announced at GTC[^4]. |
| CalVer adopted | ~2018 | Releases move to `YY.MM` cadence aligned across RAPIDS. |
| cudf.pandas | 2023 | Zero-code-change pandas accelerator with CPU fallback introduced[^5]. |
| cudf-polars | 2024 | GPU execution engine for the Polars lazy API. |

## References

[^1]: rapidsai/cudf repository, created 2017-05-07. https://github.com/rapidsai/cudf
[^2]: cuDF README — component breakdown (libcudf, pylibcudf, cudf, cudf.pandas, cudf-polars, dask-cudf). https://github.com/rapidsai/cudf/blob/main/README.md
[^3]: RAPIDS Installation Guide — system requirements and CUDA-version-matched wheels. https://docs.rapids.ai/install/
[^4]: NVIDIA, "NVIDIA Introduces RAPIDS Open-Source GPU-Acceleration Platform" — GTC Europe, 2018-10. https://rapids.ai/
[^5]: cudf.pandas documentation — zero-code-change pandas accelerator. https://docs.rapids.ai/api/cudf/stable/cudf_pandas/

## Tags

python, cpp, cuda, gpu, dataframe, pandas, apache-arrow, data-science, rapids, etl, columnar
