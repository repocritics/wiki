# astropy/astropy

> The core Python library for astronomy — units, coordinates, time, FITS I/O, and tables that nearly every astronomy package builds on.

[GitHub repo](https://github.com/astropy/astropy) ·
[Official website](https://www.astropy.org) ·
[License: BSD-3-Clause](https://github.com/astropy/astropy/blob/main/LICENSE.rst)

## Overview

Astropy began in 2011 as a consolidation effort: the astronomy community had a pile of overlapping single-purpose packages (PyFITS, PyWCS, asciitable, ATpy, vo — several maintained by the Space Telescope Science Institute) and decided to merge them into one coherent core library with shared conventions[^1]. That founding act still defines the project: `astropy.io.fits` is PyFITS's descendant, `astropy.wcs` wraps the same WCSLIB, and the package's job is to be the common substrate — units, coordinates, time, tables, file formats — that domain packages (photometry, spectroscopy, solar physics) build on rather than a complete analysis suite.

Its ~5.2k stars badly undercount its reach. Astropy is a hard dependency of essentially the whole astronomy Python stack (astroquery, photutils, specutils, sunpy, the JWST and Rubin pipelines), and the project's citation papers[^1][^2] have tens of thousands of academic citations. It is governed as a NumFOCUS-sponsored community project with a formal coordination committee, not a vendor product.

The defining tension is correctness versus overhead. Astropy chooses rigor everywhere: `Quantity` carries units through every arithmetic operation, `Time` stores two 64-bit floats per value to hold sub-nanosecond precision over astronomical timescales, and coordinate transforms route through a validated frame graph backed by the ERFA C library (a relicensed derivative of the IAU's SOFA)[^3]. The price is Python-level dispatch cost on every operation and an API that is heavier than raw NumPy. For science code where a silent unit error invalidates a paper, that trade is usually right; for hot loops it requires deliberate escape hatches.

## Getting Started

```bash
pip install astropy          # or: conda install -c conda-forge astropy
```

```python
from astropy import units as u
from astropy.coordinates import SkyCoord
from astropy.table import Table

# Units propagate through arithmetic; incompatible units raise
d = 4.2 * u.lyr
print(d.to(u.pc))                     # 1.2877 pc

# Coordinates: vectorized frame transformations
c = SkyCoord(ra=[10.68, 83.82] * u.deg, dec=[41.27, -5.39] * u.deg,
             frame="icrs")
print(c.galactic)                     # ICRS -> Galactic via the frame graph

# Unified I/O: one read/write API across FITS, ECSV, VOTable, HDF5, ...
t = Table.read("catalog.ecsv")
t.write("catalog.fits", overwrite=True)
```

## Architecture / How It Works

Astropy is a monorepo of subpackages with a few load-bearing core types:

- **`units` / `Quantity`** — `Quantity` subclasses `numpy.ndarray` and intercepts ufuncs to propagate units. Unit conversion is a graph search over defined equivalencies (e.g. spectral: wavelength ↔ frequency ↔ energy). Because it is an ndarray subclass rather than a wrapper, most NumPy functions work, but third-party code doing `np.asarray()` silently strips units — a long-standing interop hazard.
- **`coordinates` / `SkyCoord`** — a registry of reference frames (ICRS, FK5, Galactic, AltAz, GCRS, ...) connected by a transform graph; `SkyCoord` finds a path through the graph and composes the transforms. Precession/nutation/Earth-orientation math is delegated to ERFA via the `pyerfa` binding, split out of the core in the 4.x era[^3].
- **`time`** — `Time` uses a two-double (`jd1`, `jd2`) Julian date representation, so a single array keeps ~0.1 ns precision across centuries. Scale conversions (UTC ↔ TAI ↔ TT ↔ TDB) go through ERFA; UTC↔UT1 requires measured Earth-rotation data from IERS tables (see Production Notes).
- **`io.fits`** — the PyFITS lineage, with its own C code for tile compression. Header-data-unit model, memory-mapped by default. `io` also provides a unified registry so `Table.read`/`write` dispatch by format across FITS, ASCII/ECSV, VOTable, HDF5, and Parquet.
- **`table`** — column-oriented tables whose "mixin columns" can be full `Quantity`, `Time`, or `SkyCoord` objects, preserving semantics that flat NumPy or pandas columns lose. ECSV, an Astropy-designed format, round-trips those types losslessly through plain text.
- **`wcs`** — wraps Mark Calabretta's WCSLIB for pixel↔sky mappings, plus SIP and lookup-table distortions.
- Higher layers: `modeling` (composable fittable models), `cosmology`, `stats`, `convolution`, `timeseries`, `visualization` (including WCS-aware Matplotlib axes).

Deliberately out of scope: instrument-specific reduction, photometry, spectroscopy — those live in "coordinated packages" (photutils, specutils, reproject, ccdproc, astroquery) that share Astropy's release and quality conventions but version independently. The core stays comparatively slow-moving and API-stable as a result.

## Production Notes

- **The IERS download footgun.** Converting to UT1 or transforming to Earth-fixed frames (e.g. AltAz for telescope pointing) needs up-to-date IERS Earth-orientation tables. Astropy historically fetched these over HTTP at runtime and cached them in `~/.astropy` — which fails on air-gapped clusters, in containers without writable homes, and in parallel jobs racing on the cache. Newer releases ship a snapshot via the `astropy-iers-data` package (updated frequently on PyPI); pin and refresh it in offline deployments, and configure `astropy.utils.iers` (`auto_download`, `iers_degraded_accuracy`) explicitly rather than letting jobs die on a network timeout[^4].
- **Quantity overhead.** Unit tracking costs roughly microseconds per operation regardless of array size, which dominates for scalars and small arrays in tight loops. The idiom is to validate units at function boundaries (`@u.quantity_input`), then compute on `.to_value(unit)` plain arrays inside.
- **Vectorize coordinates or suffer.** Constructing one `SkyCoord` per object in a Python loop is orders of magnitude slower than one `SkyCoord` holding arrays; the frame-transform setup cost is paid per call, not per element. The same applies to `Time`. Most "astropy is slow" reports reduce to this.
- **`fits.open` defaults.** Files are memory-mapped and lazily loaded; data accessed after the `with` block closes can raise or return garbage depending on platform. Use `memmap=False` or copy arrays out when lifetimes are unclear. Cloud-hosted FITS can be read lazily over `fsspec`/S3 in recent releases.
- **pandas interop is lossy.** `Table.to_pandas()` drops units and converts `Time`/`SkyCoord` mixins; round-trips are not faithful. Keep the Astropy `Table` as the source of truth and export late.
- **Release/LTS policy.** Two releases per year, with designated LTS versions (e.g. 5.0, 6.0) receiving roughly two years of bugfixes[^5]. Pipelines pin to LTS; the NumPy 2.0 transition (astropy 6.1 era) was the most recent ecosystem-wide pinning exercise.

## When to Use / When Not

**Use when:**
- You are doing any astronomy in Python — it is the shared substrate, and fighting it means fighting every downstream package.
- You need unit-safe computation and provenance-grade time handling (observatory operations, ephemerides, archival pipelines).
- You need standards-compliant FITS/WCS/VOTable I/O without writing to the specs yourself.

**Avoid when:**
- You only need generic dataframes or arrays — pandas/Polars/NumPy are faster and lighter when units and sky geometry are irrelevant.
- You need unit checking outside astronomy — a general-purpose units library is a smaller dependency.
- You are in a microsecond-budget hot path — use Astropy at the boundaries, compiled code (or plain NumPy) inside.
- You need satellite/planetary ephemeris convenience above all — dedicated positional-astronomy libraries have a simpler API for that one job.

## Alternatives

- skyfielders/python-skyfield — use instead when you mainly need planet/star/satellite positions from ephemerides with a small, pure-Python API rather than a full framework.
- brandon-rhodes/pyephem — legacy C-based positional astronomy; only for maintaining old code, its own docs point new users to Skyfield.
- hgrecco/pint — use instead when you need physical units in non-astronomy code without pulling in the astronomy stack.
- pandas-dev/pandas — use instead for general tabular analytics where units, FITS, and sky coordinates do not matter.
- sunpy/sunpy — not a replacement but the solar-physics analog; it builds on Astropy and shows the intended layering.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2 | 2013-02 | First coherent release, accompanying the founding paper[^1]. |
| 1.0 | 2015-02 | First LTS release. |
| 2.0 | 2017-07 | Last series supporting Python 2. |
| 3.0 | 2018-02 | Python 3 only. |
| 4.0 | 2019-12 | LTS; ERFA bindings split into the separate `pyerfa` package in the 4.x era[^3]. |
| 5.0 | 2021-11 | LTS; cosmology subpackage overhaul, Parquet table I/O. |
| 6.0 | 2023-11 | LTS; IERS tables moved to the `astropy-iers-data` package[^4]. |
| 7.0 | 2024-11 | Current major series as of mid-2026. |

## References

[^1]: Astropy Collaboration, "Astropy: A community Python package for astronomy" — A&A 558, A33 (2013). https://doi.org/10.1051/0004-6361/201322068
[^2]: Astropy Collaboration, "The Astropy Project: Sustaining and Growing a Community-oriented Open-source Project" — ApJ 935, 167 (2022). https://doi.org/10.3847/1538-4357/ac7c74
[^3]: pyerfa — Python bindings for ERFA (Essential Routines for Fundamental Astronomy). https://github.com/liberfa/pyerfa
[^4]: Astropy docs, "IERS data access" and the astropy-iers-data package. https://docs.astropy.org/en/stable/utils/iers.html
[^5]: Astropy release and LTS policy (APE 2 release cycle). https://docs.astropy.org/en/stable/development/releasing.html

## Tags

python, astronomy, astrophysics, scientific-computing, units, coordinates, fits, time-series, data-tables, numpy, library
