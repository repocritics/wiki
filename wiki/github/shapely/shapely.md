# shapely/shapely

> Python bindings to the GEOS geometry engine — planar vector geometry manipulation, with a scalar object API and vectorized NumPy ufuncs.

[GitHub repo](https://github.com/shapely/shapely) ·
[Official website](https://shapely.readthedocs.io/en/stable/) ·
[License: BSD-3-Clause](https://github.com/shapely/shapely/blob/main/LICENSE.txt)

## Overview

Shapely is a Python library for the manipulation and analysis of geometric
objects in the Cartesian plane — points, lines, polygons, and their
collections — plus the predicates and operations that relate them (intersects,
contains, buffer, union, distance, and so on)[^1]. It is a thin, well-worn
wrapper over GEOS, the C++ geometry library that also powers PostGIS and is
itself a port of the Java Topology Suite (JTS)[^2]. Shapely does not implement
geometry algorithms itself; its value is a Pythonic, NumPy-aware surface over
GEOS's battle-tested topology code.

The library is deliberately narrow. It works in a flat 2D (optionally 2.5D with
a Z ordinate) Cartesian plane and has no concept of coordinate reference
systems, geodesic math, or file formats. It will happily compute the "area" of
a polygon whose coordinates are longitude/latitude degrees and return a
meaningless number — CRS awareness is left to `pyproj`, projection-aware area to
`geopandas`, and file I/O to `fiona`/`pyogrio`. This separation is intentional:
Shapely is the geometry kernel that those higher-level GIS packages build on,
most notably as the per-row geometry type inside GeoPandas.

The defining event in Shapely's history is the **2.0 rewrite** (December
2022)[^3], which absorbed the PyGEOS project. Before 2.0, Shapely bound GEOS
through `ctypes` and operated one geometry at a time; 2.0 replaced that with a C
extension exposing vectorized ufuncs, made geometries immutable and hashable,
and removed a long list of deprecated APIs. Code written for 1.x often needs
changes to run on 2.x — this is the central tension for anyone maintaining a
Shapely-dependent codebase.

## Getting Started

```bash
pip install shapely
# or, with a conda toolchain that bundles a matching GEOS:
conda install shapely --channel conda-forge
```

```python
from shapely import Point, box
import shapely
import numpy as np

# Scalar object API
patch = Point(0.0, 0.0).buffer(10.0)   # ~circular polygon
print(patch.area)                       # 313.65...  (32-segment approximation)

# Vectorized ufunc API — the 2.0 addition
geoms = np.array([Point(0, 0), Point(1, 1), Point(2, 2)])
polygon = box(0, 0, 2, 2)
print(shapely.contains(polygon, geoms))  # array([False, True, False])
```

Both styles call the same GEOS code. The scalar `.buffer()` / `.contains()`
methods loop in the Python interpreter; the `shapely.*` module-level functions
broadcast over arrays with the loop in C, and are much faster for bulk work.

## Architecture / How It Works

Shapely is a bindings layer, so most of its "internals" are really GEOS. The
Python-side structure that matters:

- **Geometry types** — `Point`, `LineString`, `LinearRing`, `Polygon`,
  `MultiPoint`, `MultiLineString`, `MultiPolygon`, `GeometryCollection`. Each
  wraps a pointer to a GEOS geometry. Since 2.0 these objects are **immutable**
  (you cannot mutate coordinates in place) and **hashable**, so they can be dict
  keys and NumPy object-array elements.
- **The C extension** — 2.0 replaced the old `ctypes` bindings with a compiled
  extension (inherited from PyGEOS). This is why `import shapely` needs a
  matching GEOS at build/install time, and why prebuilt wheels bundle their own
  GEOS.
- **Ufuncs** — module-level functions (`shapely.intersects`, `shapely.union`,
  `shapely.distance`, ...) are true NumPy universal functions: they broadcast,
  accept and return object arrays of geometries, and release the **GIL** during
  the GEOS call, so they parallelize across threads[^4].
- **STRtree** — `shapely.STRtree` is a bulk-loaded Sort-Tile-Recursive spatial
  index for fast candidate lookups (`query`, `nearest`) over large geometry
  arrays. It is the standard tool for "find geometries near/overlapping X"
  without an O(n²) scan.
- **Serialization** — WKT and WKB in/out (`shapely.to_wkt`, `from_wkb`, and the
  `shapely.wkt` / `shapely.wkb` modules), plus GeoJSON-style mapping via
  `shapely.geometry.mapping` / `shape`. There is no file reading; those helpers
  only move between in-memory representations.

Because the geometry math is GEOS, correctness and performance characteristics
are GEOS's. Shapely's job is memory management of GEOS handles, NumPy
integration, and a stable Python API surface.

## Production Notes

**GEOS coupling is the recurring operational issue.** Prebuilt pip wheels and
conda-forge packages ship a bundled GEOS, so most users never see it. But if you
build from source, or install a distro/system Shapely linked against a system
`libgeos_c`, a version mismatch between the GEOS Shapely was compiled against
and the one loaded at runtime produces import failures or subtle behavior
differences. Pin your install method; do not mix pip and system GEOS on the same
interpreter. Shapely 2.2 requires GEOS >= 3.10, Python >= 3.11, NumPy >= 1.23[^1].

**Invalid geometries and topology exceptions.** GEOS operates on
floating-point coordinates and raises `TopologyException` (surfaced as
`shapely.errors.GEOSException`) on self-intersecting polygons and other invalid
inputs. Check with `.is_valid`, repair with `shapely.make_valid()` (preferred)
or the old `buffer(0)` trick. Robustness at coordinate-scale extremes is a GEOS
concern; `shapely.set_precision()` can snap coordinates to a grid to avoid
sliver artifacts.

**Scalar vs vectorized performance.** A Python `for` loop calling `.intersects()`
on thousands of geometries is orders of magnitude slower than one
`shapely.intersects(a, b)` call over arrays. The single most common performance
mistake is iterating in Python instead of vectorizing. For spatial joins, build
an `STRtree` rather than nesting predicate loops.

**The 1.x → 2.0 migration.** The most disruptive upgrade. Breaking changes
include: geometries became immutable (no coordinate assignment); the `.ctypes`,
`.array_interface()`, and `__geo_interface__`-adjacent internals were removed;
multi-part geometry iteration and `.geoms` access changed; and numerous
deprecated aliases (`cascaded_union` → `unary_union`, `shapely.ops` reshuffles)
were dropped. Libraries downstream of Shapely (GeoPandas in particular) gated
their own 2.0 support carefully. Read the 2.0 migration guide before bumping[^3].

**No thread-safety footguns beyond GEOS.** Because operations release the GIL
and geometries are immutable since 2.0, read-only concurrent use across threads
is safe and genuinely parallel — a rare property for a CPython library.

## When to Use / When Not

**Use when:**
- You need 2D vector geometry predicates and operations in Python (GIS, CAD-like
  work, computational geometry, spatial feature engineering).
- You are already in the NumPy/pandas world and want vectorized geometry ops.
- You are building on top of it via GeoPandas and want the underlying geometry
  primitives directly.

**Avoid when:**
- You need coordinate reference systems, projections, or geodesic (on-the-globe)
  distance and area — Shapely is planar and CRS-blind; use pyproj/GeoPandas.
- You need to read or write spatial files (Shapefile, GeoPackage, GeoJSON files)
  — that is fiona/pyogrio/GDAL territory.
- You need 3D solids or true volumetric geometry — Shapely's Z is carried but
  most operations are strictly 2D.
- You want a pure-Python dependency with no native library — Shapely always
  needs GEOS.

## Alternatives

- geopandas/geopandas — use instead when you want geometry attached to tabular
  data with CRS support; it wraps Shapely as its geometry column.
- Toblerity/Fiona (and geopandas/pyogrio) — use instead when the task is reading
  and writing vector data files, which Shapely deliberately does not do.
- pyproj4/pyproj — use instead (or alongside) when you need coordinate reference
  systems, transforms, and geodesic calculations.
- libgeos/geos — use directly from C/C++ when you want the engine without the
  Python layer; Shapely is its most-used binding.
- pygeos/pygeos — historical: the vectorized predecessor that was merged into
  Shapely 2.0 and is now archived; new code should use Shapely, not PyGEOS.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010 | Early stable ctypes-based bindings to GEOS. |
| 1.8.0 | 2021-10 | Last major line of the 1.x series; deprecations flagged for 2.0. |
| 2.0.0 | 2022-12 | Rewrite absorbing PyGEOS: C extension, vectorized ufuncs, immutable/hashable geometries, many removals[^3]. |
| 2.1.0 | 2025 | Continued 2.x line; API refinements and GEOS version bumps. |
| 2.2 | 2026 | Requires Python >= 3.11, GEOS >= 3.10, NumPy >= 1.23[^1]. |

## References

[^1]: Shapely README and documentation. https://shapely.readthedocs.io/en/stable/
[^2]: GEOS — Geometry Engine Open Source. https://libgeos.org/
[^3]: Shapely 2.0 release and migration guide. https://shapely.readthedocs.io/en/stable/migration.html
[^4]: Shapely user manual, on GIL release and multithreading. https://shapely.readthedocs.io/en/stable/manual.html

## Tags

python, geometry, gis, geospatial, geos, computational-geometry, vector-geometry, numpy, bindings, spatial-index
