# numpy/numpy

> The N-dimensional array that the rest of the Python scientific stack is built on top of.

[GitHub repo](https://github.com/numpy/numpy) ·
[Official website](https://numpy.org) ·
[License: BSD-3-Clause](https://github.com/numpy/numpy/blob/main/LICENSE.txt)

## Overview

NumPy provides `ndarray`, a strided, homogeneously-typed, contiguous-in-memory
array, plus the vectorized operations, broadcasting rules, and dtype system that
turn Python from a slow scripting language into a usable numerical one. It began
in 2005 when Travis Oliphant unified the two competing array libraries of the
time — Numeric (Jim Hugunin, 1995) and Numarray — into a single package, with
NumPy 1.0 following in 2006[^1]. Nearly the entire PyData ecosystem — pandas,
scikit-learn, SciPy, matplotlib, and the array-producing halves of PyTorch, JAX,
and TensorFlow — either builds on the `ndarray` or copies its interface.

The defining tension is that NumPy is simultaneously a high-level array library
and a low-level ABI contract. The Python API (`np.array`, broadcasting, fancy
indexing) is stable and pleasant; the C-API that compiled extensions link
against is a hard dependency that ties thousands of downstream wheels to NumPy's
binary layout. This is why the 2.0 release in 2024 — the first ABI break in
roughly a decade — was a coordinated, ecosystem-wide event rather than a routine
upgrade[^2].

NumPy is a NumFOCUS-sponsored, community-run project with no single corporate
owner. It is maintained conservatively: correctness and backwards compatibility
are prioritized over feature velocity, and the reference-implementation status
of the array-programming model is treated as a responsibility.

## Getting Started

```bash
pip install numpy          # or: conda install numpy / uv add numpy
```

```python
import numpy as np

a = np.arange(12).reshape(3, 4)      # 3x4 int array, one contiguous buffer
b = a.mean(axis=0)                    # column means -> shape (4,)
mask = a > 5
a[mask] = 0                          # boolean (fancy) indexing, in place

# broadcasting: (3,4) * (4,) -> (3,4), no explicit loop, no copy of `a`
scaled = a * np.array([1.0, 0.5, 0.25, 0.0])
```

Prefer the modern random API over the legacy global one:

```python
rng = np.random.default_rng(seed=42)   # Generator, not RandomState
samples = rng.normal(size=1_000)
```

## Architecture / How It Works

An `ndarray` is a thin Python object wrapping four things: a pointer to a raw
data buffer, a `dtype` describing element type and byte width, a `shape` tuple,
and a `strides` tuple giving the byte step along each axis. Because indexing is
pure stride arithmetic, transposes, slices, and reshapes usually return *views*
that share the underlying buffer — no data is copied. This is the source of both
NumPy's speed and its most common bug class (mutating a view mutates the
original).

Element-wise operations are implemented as **ufuncs** (universal functions):
compiled C loops that iterate over the buffer, with broadcasting handled by the
iterator (`nditer`) that virtually stretches size-1 axes without materializing
them. Most ufunc inner loops are single-threaded C; NumPy accelerates them where
it can with SIMD via a "universal intrinsics" layer that dispatches at runtime to
the widest instruction set the CPU supports (SSE/AVX/AVX-512 on x86, NEON/SVE on
ARM)[^3].

Linear algebra (`np.linalg`, matrix multiply via `@`) is *not* NumPy's own code
— it delegates to a BLAS/LAPACK backend. The official wheels bundle OpenBLAS;
conda builds often use MKL; macOS can use Accelerate. Which backend is linked
determines both performance and, occasionally, numerical results at the last
ULP. This delegation means `a @ b` is multithreaded (via BLAS) while `a + b` is
not, a frequent source of confusion about NumPy's threading behavior.

The build system moved from `distutils`/`setuptools` to **Meson** in the 2.0
cycle, forced partly by `distutils`' removal from the Python standard library[^2].
The random module was re-architected in 1.17 around a pluggable `BitGenerator` +
`Generator` design (`default_rng`), leaving the old `RandomState` frozen for
reproducibility[^4].

## Production Notes

**The 2.0 upgrade is the big one.** NumPy 2.0 (June 2024) changed the C-ABI,
removed long-deprecated aliases (`np.float`, `np.int`, `np.bool`, `np.object`,
`np.str`), moved and renamed many attributes, and — via NEP 50 — changed scalar
type-promotion rules so that operations like `np.float32(3) + 3.0` no longer
silently upcast based on values[^2][^5]. Pure-Python code often needs edits;
compiled extensions must be rebuilt against 2.0 to load under it. Build wheels
against NumPy 2.x and they remain importable under 1.x, but not the reverse.

**Silent integer overflow.** Fixed-width integer dtypes wrap around without
warning: `np.int8(127) + np.int8(1)` yields `-128`. There is no automatic
promotion to a wider type or to Python's arbitrary-precision int.

**Views vs. copies are invisible at the call site.** Whether an operation
returns a view or a copy depends on the operation and memory layout, not the
syntax. `a[::2]` is a view; `a[[0, 2]]` (fancy indexing) is a copy. Assigning
into what you thought was a copy silently mutates shared data.

**BLAS thread oversubscription.** NumPy's BLAS backend spawns its own thread
pool. Combined with an outer `multiprocessing`/`joblib` pool this oversubscribes
cores and *slows* work down. Pin it with `OMP_NUM_THREADS` /
`OPENBLAS_NUM_THREADS` (or `threadpoolctl`) in multi-process pipelines.

**Object and string arrays give up the point.** `dtype=object` arrays fall back
to per-element Python calls and are no faster than lists. A dedicated
variable-width string dtype (`StringDType`) landed in 2.0 to address the classic
fixed-width `U`/`S` pain, but it is newer and less battle-tested[^2].

**Free-threading (no-GIL).** Recent releases (2.1+) add experimental support for
the free-threaded CPython 3.13t builds; it works but is not yet a performance
recommendation for production[^6].

## When to Use / When Not

**Use when:**
- You need in-memory, CPU, dense numerical arrays and the vast library that
  assumes them (pandas, scikit-learn, SciPy).
- You want a stable, vendor-neutral array interface as the lingua franca between
  tools.
- Data fits comfortably in RAM and fits a homogeneous dtype.

**Avoid / add something else when:**
- You need GPU/TPU acceleration or autodiff — reach for JAX, PyTorch, or CuPy.
- Data exceeds memory or must be distributed — use Dask or out-of-core tooling.
- Your data is tabular/heterogeneous with labels — pandas or Polars sit on or
  beside NumPy for that.
- You need sparse matrices — that lives in SciPy, not NumPy.

## Alternatives

- cupy/cupy — near drop-in NumPy API that runs on NVIDIA/AMD GPUs; use when the
  same array code needs GPU execution.
- google/jax — NumPy-compatible API plus autodiff and XLA compilation; use for
  ML/research with gradients and accelerators.
- pytorch/pytorch — tensors with autograd and GPU; use when deep learning is the
  primary workload, not general array math.
- dask/dask — chunked, lazy, parallel arrays mirroring the NumPy API; use when
  data is larger than memory or spread across a cluster.
- pydata/xarray — labeled, N-dimensional arrays built on NumPy; use when axes
  carry named coordinates (time, lat/lon, etc.).

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2006-10 | First release; unified Numeric + Numarray into one `ndarray`[^1]. |
| 1.7 | 2013-02 | Long deprecation cycle toward a cleaner API begins. |
| 1.17 | 2019-07 | New `Generator`/`BitGenerator` random API (`default_rng`)[^4]. |
| 1.20 | 2021-01 | Type annotations and `numpy.typing` added. |
| 1.25 | 2023-06 | Last stable line before the 2.0 transition. |
| 2.0.0 | 2024-06-16 | C-ABI break, NEP 50 promotion, alias removals, Meson build, `StringDType`[^2][^5]. |
| 2.1 | 2024-08 | Python 3.13 support; experimental free-threading[^6]. |
| 2.3 | 2025-06 | Continued 2.x line; further free-threaded and dtype work. |

## References

[^1]: NumPy history / "About NumPy" — origins in Numeric (1995) and Numarray, unified by Travis Oliphant. https://numpy.org/doc/stable/dev/index.html
[^2]: "NumPy 2.0.0 Release Notes" — ABI break, NEP 50, alias removals, Meson build, StringDType. https://numpy.org/doc/stable/release/2.0.0-notes.html
[^3]: NumPy CPU/SIMD optimization documentation (universal intrinsics dispatch). https://numpy.org/doc/stable/reference/simd/index.html
[^4]: "Random sampling (numpy.random)" — Generator vs. legacy RandomState. https://numpy.org/doc/stable/reference/random/index.html
[^5]: NEP 50 — "Promotion rules for Python scalars." https://numpy.org/neps/nep-0050-scalar-promotion.html
[^6]: Harris et al., "Array programming with NumPy," Nature 585 (2020). https://doi.org/10.1038/s41586-020-2649-2

## Tags

python, numerical-computing, ndarray, scientific-computing, linear-algebra, c-extension, broadcasting, blas, data-science, array-programming
