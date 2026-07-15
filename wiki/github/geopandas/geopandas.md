# geopandas/geopandas

> pandas for vector geospatial data — GeoSeries and GeoDataFrame that carry geometry columns and a coordinate reference system.

[GitHub repo](https://github.com/geopandas/geopandas) ·
[Official website](https://geopandas.org) ·
[License: BSD-3-Clause](https://github.com/geopandas/geopandas/blob/main/LICENSE.txt)

## Overview

GeoPandas extends pandas with two types: `GeoSeries` and `GeoDataFrame`, subclasses of `pandas.Series` and `pandas.DataFrame` that hold a geometry column alongside ordinary tabular columns[^1]. Geometry values are shapely objects; the geometric predicates and operations (intersection, buffer, distance, spatial joins) are delegated to shapely/GEOS, coordinate transforms to pyproj/PROJ, and file I/O to GDAL via pyogrio. GeoPandas itself is the glue: it makes "a table with a spatial column" behave like a normal DataFrame while keeping the geometry and CRS coherent.

The project has been fiscally sponsored by NumFOCUS under an open governance model since it matured past its single-author phase[^2]. It is the default in-memory vector container in the scientific Python geo stack, and most higher-level tools (contextily, folium bindings, dask-geopandas, movingpandas, osmnx) either return GeoDataFrames or consume them.

The defining tension is the native-dependency chain. GeoPandas is pure Python, but its runtime is a stack of C libraries — GEOS, PROJ, GDAL — surfaced through shapely, pyproj, and pyogrio. Getting those to agree on versions has historically been the hardest part of using the library, which is why conda-forge, not pip, has long been the recommended install path[^3]. The second tension is that all operations are planar (cartesian): GeoPandas stores a CRS but does not reproject for you, so `area` and `distance` on lat/long data return numerically meaningless values unless you project first.

## Getting Started

```bash
# Recommended: conda-forge pulls compatible GEOS/PROJ/GDAL binaries
conda install -c conda-forge geopandas
# pip works when wheels are available for your platform
pip install geopandas
```

```python
import geopandas
from shapely.geometry import Point

# Read any GDAL/OGR vector format (shapefile, GeoPackage, GeoJSON, ...)
gdf = geopandas.read_file("nybb.gpkg")

# Geometry-aware operations return GeoPandas or pandas objects
gdf["centroid"] = gdf.geometry.centroid
gdf = gdf.to_crs(epsg=4326)              # reproject to WGS84 (lat/long)

# Spatial predicate query + spatial join
pts = geopandas.GeoDataFrame(
    geometry=[Point(-73.98, 40.75)], crs="EPSG:4326"
)
joined = geopandas.sjoin(pts, gdf, predicate="within")
gdf.to_parquet("boroughs.parquet")       # GeoParquet output
```

## Architecture / How It Works

A `GeoDataFrame` is an ordinary pandas DataFrame with one designated "active" geometry column (`gdf.geometry`), though multiple geometry columns can coexist. The geometry column is backed by a `GeometryArray`, a pandas `ExtensionArray` that stores geometries in a contiguous NumPy object array and dispatches vectorized operations to shapely 2.0's array interface[^4]. Each vector operation (`buffer`, `intersection`, `contains`) is a single call into GEOS over the whole array rather than a Python loop over individual shapely objects — this vectorization is the main reason modern GeoPandas is usable on non-trivial datasets.

The CRS lives as metadata on the `GeometryArray` (a `pyproj.CRS` object). It travels with the data and is written to/read from files, but it is advisory: GeoPandas does not enforce that two operands share a CRS, and does not auto-reproject. `to_crs()` is the explicit reprojection step, backed by pyproj/PROJ transformation pipelines.

I/O goes through **pyogrio** by default as of 1.0[^5], a vectorized GDAL/OGR binding that reads whole layers into Arrow/NumPy buffers. The older **fiona** engine (row-by-row iteration over OGR) is still selectable via `engine="fiona"`. Columnar formats have a separate path: `read_parquet`/`to_parquet` and the Feather equivalents implement the **GeoParquet** specification on top of Apache Arrow, encoding geometry as WKB with CRS stored in file metadata[^6].

Spatial joins and `overlay` use a spatial index — shapely 2.0's GEOS `STRtree`, with `rtree` (libspatialindex) as an alternative backend — to prune candidate pairs before running exact predicates. Plotting is delegated to matplotlib and is intentionally basic; interactive maps (`.explore()`) render through folium/leaflet.

## Production Notes

**Installation is the historic footgun.** Mixing a pip-installed GDAL against a conda GEOS, or two shapely builds linking different GEOS versions, produces import-time crashes or silent geometry corruption. Pin the whole geo stack from one channel (conda-forge) or rely on the manylinux wheels, and do not mix. This has improved substantially with wheels but remains the first thing to check when something segfaults.

**CRS mistakes are silent.** Because operations are planar and no CRS is enforced, computing `.area` or `.distance` on EPSG:4326 (degrees) returns numbers in square-degrees / degrees — plausible-looking and wrong. Project to an appropriate equal-area or local UTM CRS first, or use the geodesic helpers. Spatially joining two layers in different CRSs raises now, but arithmetic between mismatched geometries may not.

**Everything is in memory, single-process.** A GeoDataFrame holds all geometries and attributes in RAM, and most operations run on one core. Large dissolves, `overlay`, and all-pairs `sjoin` are the usual performance cliffs. For datasets beyond memory or when parallelism is needed, dask-geopandas partitions a GeoDataFrame across Dask workers with a mostly-compatible API.

**The 1.0 / shapely 2.0 transition was a real break.** GeoPandas 1.0 (2024) removed the optional **pygeos** backend and requires shapely ≥ 2.0[^7]. Code written against 0.8–0.13 that imported pygeos, relied on fiona-specific behavior, or used since-deprecated methods (`GeoSeries.geom_type` spellings, `unary_union` vs `union_all`, retired plotting kwargs) needs updating. Reading files also changed default engine to pyogrio, which differs from fiona in edge cases (field type coercion, handling of null geometries, layer options).

**`sjoin` semantics changed** across versions: the `op=` keyword became `predicate=`, and index/column naming after joins has shifted, so pinning a GeoPandas version in reproducible pipelines matters more than the loose "any 0.x" that many older tutorials assume.

## When to Use / When Not

**Use when:**
- You have vector data (points/lines/polygons) that fits in memory and want pandas ergonomics for it.
- You need file interchange across GIS formats (Shapefile, GeoPackage, GeoJSON, GeoParquet, PostGIS).
- You want spatial joins, overlays, dissolves, buffering, and reprojection with a tabular API.
- You are already in the scientific Python stack and want interop with pandas, matplotlib, and Arrow.

**Avoid when:**
- Your data is raster (imagery, DEMs, gridded climate) — use rasterio / rioxarray / xarray instead.
- The data does not fit in RAM or you need cluster-scale parallelism — reach for dask-geopandas or a spatial database.
- You only manipulate individual geometries with no tabular/CRS/I/O layer — shapely alone is lighter.
- You need heavy spatial statistics/econometrics — that lives in the PySAL family, which consumes GeoDataFrames.

## Alternatives

- shapely/shapely — use when you only need geometry construction and predicates, without the DataFrame, CRS, or I/O layer.
- geopandas/dask-geopandas — use when datasets exceed memory or you need multi-core / out-of-core processing with a near-identical API.
- pysal/pysal — use when the goal is spatial statistics, weights, and econometrics rather than data wrangling.
- rasterio/rasterio — use when your data is raster rather than vector geometry.
- geopolars/geopolars — use when you want polars-backed geospatial dataframes and can tolerate an experimental, incomplete API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2014 | First PyPI release; GeoSeries/GeoDataFrame on pandas + shapely + fiona. |
| 0.8.0 | 2020-06 | Optional pygeos backend for vectorized geometry ops; large speedups[^8]. |
| 0.11.0 | 2022-06 | GeoParquet I/O maturing; CRS handling on pyproj CRS objects. |
| 0.12.0 | 2022-10 | shapely 2.0 support alongside pygeos. |
| 0.14.0 | 2023-09 | pygeos deprecated in favor of shapely 2.0's vectorized API. |
| 1.0.0 | 2024-06 | pygeos removed, shapely ≥ 2.0 required, pyogrio default I/O engine[^7]. |

## References

[^1]: GeoPandas documentation, "Introduction / Data Structures". https://geopandas.org/en/stable/docs/user_guide/data_structures.html
[^2]: GeoPandas governance model, NumFOCUS fiscal sponsorship. https://github.com/geopandas/governance/blob/main/Governance.md
[^3]: GeoPandas installation guide (conda-forge recommended). https://geopandas.org/en/stable/getting_started/install.html
[^4]: shapely 2.0 vectorized (NumPy) interface, used by GeoPandas GeometryArray. https://shapely.readthedocs.io/en/stable/
[^5]: pyogrio — vectorized GDAL/OGR vector I/O. https://pyogrio.readthedocs.io/
[^6]: GeoParquet specification. https://geoparquet.org/
[^7]: GeoPandas 1.0 release notes / changelog. https://geopandas.org/en/stable/docs/changelog.html
[^8]: GeoPandas 0.8.0 changelog (pygeos backend). https://geopandas.org/en/stable/docs/changelog.html

## Tags

python, geospatial, gis, pandas, vector-data, shapely, gdal, coordinate-reference-system, geoparquet, dataframe, spatial-join, numfocus
