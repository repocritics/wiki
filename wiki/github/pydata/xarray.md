# pydata/xarray

> N-dimensional labeled arrays for Python — pandas' data model generalized past two dimensions, with netCDF and dask underneath.

[GitHub repo](https://github.com/pydata/xarray) ·
[Official website](https://xarray.dev) ·
[License: Apache-2.0](https://github.com/pydata/xarray/blob/main/LICENSE)

## Overview

Xarray adds labels — named dimensions, coordinates, and attributes — on top of
NumPy-like arrays, so that operations reference `time` and `lat` by name instead
of axis integers[^1]. It began in 2014 as "xray", an internal tool at The Climate
Corporation written by Stephan Hoyer and colleagues, was renamed xarray in
January 2016, and became a NumFOCUS fiscally sponsored project in August 2018[^2].
Its data model is borrowed almost directly from the netCDF file format, which is
why it dominates geoscience, climate, and remote-sensing work.

The two core structures are `DataArray` (one labeled N-D array plus its
coordinates) and `Dataset` (a dict-like collection of DataArrays that share
dimensions, mirroring a netCDF file with multiple variables). Label-based
indexing (`.sel`), dimension-name reductions (`.mean("time")`), automatic
alignment on coordinates, and split-apply-combine (`.groupby`, `.resample`,
`.rolling`) are the day-to-day API. The value proposition is that metadata rides
along with the data and operations stay readable at high dimensionality.

The defining tension is that xarray is a thin, deliberately un-opinionated layer
over other people's arrays. It computes almost nothing itself: NumPy does the
in-memory math, dask does the out-of-core and parallel math, and a stack of I/O
backends does the reading and writing. That keeps xarray small and composable,
but it also means most real performance and memory behavior — and most of the
footguns — actually live in NumPy, dask, or HDF5, not in xarray.

## Getting Started

```bash
pip install "xarray[io]"        # pulls netCDF4, h5netcdf, zarr, etc.
# or: conda install -c conda-forge xarray dask netCDF4
```

```python
import numpy as np
import xarray as xr

temp = xr.DataArray(
    np.random.randn(3, 4),
    dims=("time", "location"),
    coords={"time": ["2026-01-01", "2026-01-02", "2026-01-03"],
            "location": ["a", "b", "c", "d"]},
    name="temperature",
)

# label-based selection and named-dimension reduction
print(temp.sel(location="b").mean("time"))

# split-apply-combine by a coordinate
ds = temp.to_dataset()
ds.to_netcdf("temps.nc")                 # write
again = xr.open_dataset("temps.nc")      # read (lazy for on-disk arrays)
```

## Architecture / How It Works

`DataArray` and `Dataset` wrap a lower-level `Variable` (recently factored
further into `NamedArray`, a minimal dims-plus-data container aligned with the
Python array API standard[^3]). A Variable holds the raw duck-array — this is the
critical seam: the payload can be a NumPy array, a dask array, a CuPy array, a
sparse array, or anything implementing the array API. Xarray dispatches through
`__array_function__` / `__array_ufunc__` rather than assuming NumPy, which is how
one codebase spans in-memory, GPU, and cluster execution.

Indexes are a separate subsystem. For years every dimension coordinate was
silently backed by a `pandas.Index`; the multi-year "explicit indexes" refactor
made indexes first-class and pluggable, so `.sel` can be served by a pandas
Index, a `CFTimeIndex` for non-standard calendars, or a custom (e.g. spatial /
KD-tree) index[^4]. This is powerful and still has rough edges around MultiIndex
and non-dimension coordinates.

Parallelism is delegated to dask. Calling `.chunk()` (or opening with
`chunks=...`) swaps NumPy payloads for dask arrays; every subsequent operation
builds a lazy task graph, and nothing runs until `.compute()`, `.load()`, or a
write. `xr.apply_ufunc` and `xr.map_blocks` are the escape hatches for wrapping
custom or gufunc-style kernels so they parallelize correctly.

I/O is a backend registry: netCDF4/HDF5, h5netcdf, scipy, zarr, pydap, and
third-party entrypoints (rioxarray for GDAL rasters, cfgrib for GRIB). CF
conventions decoding — times, `scale_factor`/`add_offset`, `_FillValue` → NaN — is
applied on read by default. `DataTree`, a hierarchical container of nested
Datasets modeling netCDF/HDF groups, began as the separate `datatree` package and
was merged into xarray core in the 2024.10 release[^5].

## Production Notes

**Laziness is not automatic.** Without dask, `open_dataset` is lazy only for the
on-disk array values; the moment you touch `.values`, call `.compute()`, or feed
data into a NumPy function, the whole thing loads into RAM. Teams routinely OOM by
opening a multi-GB netCDF and then doing one careless `.values` or `float(...)`.

**Chunking is the real skill.** With dask, wrong chunk sizes are the top
performance problem: chunks too small explode the task graph (millions of tasks,
scheduler overhead dominates); too large and you lose parallelism and blow memory.
Rechunking mid-pipeline is expensive. `open_mfdataset` across thousands of files
can spend most of its time just building the combine graph — pass explicit
`combine`, `concat_dim`, `coords="minimal"`, and `compat` rather than the
auto-detect defaults.

**HDF5 is not thread-safe.** The netCDF4/HDF5 backend serializes access with a
global lock; concurrent reads under a threaded dask scheduler can bottleneck or
deadlock. h5netcdf or the zarr backend (which is cloud- and parallel-friendly)
are the usual answers for concurrent workloads.

**Silent alignment surprises.** Arithmetic between two objects auto-aligns on
coordinates (inner join by default) and auto-broadcasts by dimension name. A
mismatched coordinate can quietly shrink your data to the intersection, or an
unexpected shared-but-differently-named dimension can broadcast a small array into
a huge one. Check shapes after binary ops on data from different sources.

**Versioning and churn.** Xarray switched from SemVer (`0.x`) to calendar
versioning (`YYYY.MM.MICRO`) starting with `2022.03.0`[^6]. Releases are roughly
monthly and generally additive, but the indexes and NamedArray refactors have
shifted internals; code reaching past the public API (into `Variable`,
`IndexVariable`, private index internals) breaks between releases.

## When to Use / When Not

**Use when:**
- Your data is genuinely N-dimensional with meaningful labels (time × lat × lon ×
  level), and axis-integer bookkeeping in NumPy is error-prone.
- You work with netCDF, GRIB, Zarr, or CF-convention datasets.
- You need to scale the same code from a laptop array to a dask cluster.
- Metadata and coordinates must survive slicing, aggregation, and I/O.

**Avoid when:**
- Your data is fundamentally tabular / 2-D — pandas or Polars is a better fit.
- You need raw numeric speed with no metadata overhead — call NumPy directly.
- Your arrays are ragged / jagged rather than rectangular N-D — xarray assumes
  dense rectilinear arrays.
- The labeling adds no value (pure linear algebra, ML tensor plumbing) — the
  abstraction is then just overhead.

## Alternatives

- pandas-dev/pandas — use instead when your data is 1-D/2-D tabular; xarray is
  explicitly "pandas for >2 dimensions".
- numpy/numpy — use directly when you want raw N-D arrays and don't need labels,
  alignment, or metadata.
- dask/dask — use `dask.array` directly when you want out-of-core/parallel arrays
  without the labeled-coordinate layer.
- SciTools/iris — use instead for strict Met Office / CF meteorology workflows
  where the CF data model is enforced rather than optional.
- zarr-developers/zarr-python — the chunked storage format xarray writes to; use
  it directly when you need low-level control over the on-disk array layout.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014-05 | First open-source release as "xray"[^2]. |
| — | 2016-01 | Renamed from xray to xarray[^2]. |
| — | 2018-08 | Became a NumFOCUS fiscally sponsored project[^2]. |
| 0.16 | 2020-07 | Flexible/explicit-indexes work underway; backend entrypoint API. |
| 2022.03.0 | 2022-03 | Switch to calendar versioning (`YYYY.MM.MICRO`)[^6]. |
| 2023.x | 2023 | NamedArray extraction; array-API-standard alignment[^3]. |
| 2024.10.0 | 2024-10 | `DataTree` merged into core from the datatree package[^5]. |

## References

[^1]: Xarray documentation — "Overview: Why xarray?". https://docs.xarray.dev/en/stable/getting-started-guide/why-xarray.html
[^2]: Xarray README, "History" section, and project docs. https://github.com/pydata/xarray
[^3]: Xarray docs — NamedArray and the Python array API standard. https://docs.xarray.dev/en/stable/internals/index.html
[^4]: Hoyer et al. — "Flexible indexes" design and roadmap. https://docs.xarray.dev/en/stable/internals/how-to-create-custom-index.html
[^5]: Xarray docs — DataTree / hierarchical data. https://docs.xarray.dev/en/stable/user-guide/hierarchical-data.html
[^6]: Xarray docs — "What's New" / release history (CalVer adoption). https://docs.xarray.dev/en/stable/whats-new.html

## Tags

python, n-dimensional-arrays, netcdf, numpy, pandas, dask, zarr, geoscience, scientific-computing, labeled-data, data-analysis
