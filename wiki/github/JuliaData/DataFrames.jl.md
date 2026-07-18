# JuliaData/DataFrames.jl

> In-memory tabular data for Julia — the ecosystem's de facto standard data frame, built on plain vectors and a deliberately type-unstable container.

[GitHub repo](https://github.com/JuliaData/DataFrames.jl) ·
[Official docs](https://dataframes.juliadata.org/stable/) ·
[License: MIT](https://github.com/JuliaData/DataFrames.jl/blob/main/LICENSE.md)

## Overview

DataFrames.jl is Julia's primary in-memory tabular data package, started in 2012 — before Julia 1.0 existed — by Harlan Harris, John Myles White and others, and today maintained collectively by the JuliaData organization, with Bogumił Kamiński and Milan Bouchet-Valat as the long-term stewards[^1]. It took nearly nine years to reach 1.0 (2021), a period that included several full redesigns of missing-value handling and the column API; since 1.0 the package has followed SemVer and been notably stable[^2].

Its ~1.8k stars understate its position: GitHub star counts run low across the Julia ecosystem, and within it DataFrames.jl is the assumed substrate — CSV.jl, Arrow.jl, the statistics stack (GLM.jl, MixedModels.jl), and plotting recipes all interoperate with it through the Tables.jl interface rather than depending on it directly[^3].

The defining tension is architectural: a `DataFrame` stores its columns as a `Vector{AbstractVector}`, so column element types are invisible to the compiler. This is a conscious trade — it keeps compilation latency bounded for interactive work with wide, heterogeneous tables — but it means naive row-by-row code hits dynamic dispatch, and getting native-Julia speed out of a data frame requires knowing the function-barrier idiom. DataFrames.jl is fast where operations are whole-column or go through its own kernels (joins, grouping), and slow where users write loops as if it were a typed struct.

## Getting Started

```julia
using Pkg; Pkg.add("DataFrames")
```

```julia
using DataFrames, Statistics

df = DataFrame(name=["Alice", "Bob", "Clara"],
               age=[25, 30, 35],
               dept=["eng", "ops", "eng"])

# split-apply-combine with the source => fun => dest mini-language
combine(groupby(df, :dept), :age => mean => :mean_age)

# row filtering and column transformation
subset(df, :age => a -> a .>= 30)
transform(df, [:name, :dept] => ByRow((n, d) -> "$n@$d") => :id)
```

I/O lives in sibling packages: `CSV.read("file.csv", DataFrame)` via CSV.jl, `Arrow.Table` for Arrow files, and any Tables.jl-compatible source converts with `DataFrame(source)`.

## Architecture / How It Works

A `DataFrame` is a thin mutable container: a vector of column vectors plus an `Index` mapping names to positions. Columns can be any `AbstractVector` — plain `Vector`, `PooledArray`, `CategoricalArray`, an Arrow memory-mapped vector, or a lazy `SubArray`. The container imposes no schema type parameter, which is the opposite choice from typed-table designs and the root of both its ergonomics and its performance caveats.

Key design elements:

- **Missing values** are Julia-native: a column allowing missings has element type `Union{T, Missing}`, with compiler-level union-splitting making this nearly free. This replaced the earlier `DataArrays`-based `NA` design in the 0.11 era (2017), one of the project's largest historical rewrites[^4].
- **The transformation mini-language** (`source_cols => function => dest_name`) is the API's center of gravity since 0.21 (2020): `select`, `transform`, `combine`, and `subset` all accept it, and it doubles as the split-apply-combine interface when applied to a `GroupedDataFrame`[^2].
- **Grouping** builds an index over group keys once; `combine`/`select` over groups then compile a specialized kernel per unique column-type combination. Many grouped and join operations run multithreaded by default (opt out with `threads=false`).
- **Copy vs. view semantics are explicit and subtle**: `df.col` and `df[!, :col]` return the actual column vector (mutations write through); `df[:, :col]` returns a copy; `@view` / `view(df, ...)` produce `SubDataFrame`s. Assignment follows parallel rules (`df[!, :col] = v` replaces the column, `df[:, :col] = v` mutates in place).
- **Tables.jl** is the interop boundary: DataFrames.jl consumes and produces the row/column iteration interface, so the wider ecosystem avoids a hard dependency on it[^3].
- **Metadata** (table-level and column-level key-value pairs with propagation rules) was added in 1.4 (2022), primarily for provenance and label use cases.

Companion packages provide the macro conveniences the core deliberately omits: DataFramesMeta.jl (`@rsubset`, `@rtransform`), Chain.jl (`@chain` piping).

## Production Notes

- **The function barrier is not optional.** `for row in eachrow(df)` with per-element access is dynamically dispatched and can be orders of magnitude slower than the same loop over extracted columns. Pass columns into a concrete function (`process(df.a, df.b)`), or use `Tables.namedtupleiterator` for type-stable row iteration — at the cost of compiling a specialized method per schema, which is itself expensive for very wide tables.
- **Everything is in RAM, and operations copy.** There is no out-of-core or lazy execution engine. Joins and `combine` materialize results; a join of two multi-GB frames can transiently need several times the input size. For larger-than-memory work the ecosystem answer is Arrow.jl memory-mapping or pushing the heavy query into DuckDB and materializing the result as a `DataFrame`.
- **Compilation latency.** First `groupby`/`combine`/join in a fresh session compiles kernels. Julia 1.9+ native code caching and the package's own precompile workloads reduced this substantially, but CLI-style scripts that start a new Julia process per run still pay a tax; long-lived sessions or sysimages are the standard mitigation.
- **Aliasing footgun.** Because `df.col` is the real vector, code that grabs a column and mutates it silently changes the frame — and `DataFrame(x; copycols=false)` aliases input vectors. Use `copy` deliberately at API boundaries.
- **Row-wise construction is slow.** Building tables via `push!(df, row)` in hot loops is much slower than accumulating typed vectors and constructing once at the end.
- **Threading interactions.** Default multithreaded grouped operations spawn tasks; inside already-parallel user code or when calling non-thread-safe functions in `combine`, pass `threads=false`.
- **Upgrade history is calm post-1.0.** The painful churn (wholesale renames, `by`/`aggregate` removal, missing-value redesigns) happened in the 0.x era; 1.x upgrades have been additive with deprecation cycles.

## When to Use / When Not

**Use when:**
- You are already in Julia — for simulation post-processing, statistics, econometrics, or feeding GLM.jl/MixedModels.jl — and need the standard interchange format the rest of the ecosystem expects.
- Your data fits in memory and your workload is column-oriented transformations, grouping, and joins.
- You want missing-value handling integrated with the language's type system rather than sentinel values.

**Avoid when:**
- Data exceeds RAM or you need lazy/streaming query optimization — use DuckDB or Polars-style engines instead of fighting the eager copy semantics.
- Your workload is dominated by row-wise logic over a fixed narrow schema — a `Vector` of structs or TypedTables.jl is simpler and faster.
- You need Python/R ecosystem reach; adopting Julia solely for its data frame is rarely justified on that feature alone.

## Alternatives

- pandas-dev/pandas — use instead when your team and dependency stack are Python; larger API surface, far larger ecosystem, similar in-memory eager model.
- pola-rs/polars — use instead when you need lazy query optimization, streaming, and multicore performance on larger-than-comfortable data.
- duckdb/duckdb — use instead when workloads are relational/SQL-shaped or out-of-core; also works well *with* DataFrames.jl as an execution engine.
- Rdatatable/data.table — use instead in R for terse, very fast in-memory aggregation.
- JuliaData/TypedTables.jl — use instead within Julia for narrow fixed-schema tables where full type stability matters more than dynamic columns.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-0.x | 2012-07 | Repo created; grew out of early Julia statistics packages[^1]. |
| 0.11 | 2017 | Dropped DataArrays; moved to `Union{T, Missing}` columns[^4]. |
| 0.21 | 2020 | `source => fun => dest` transformation mini-language across select/transform/combine[^2]. |
| 1.0 | 2021 | API declared stable after ~9 years of 0.x; SemVer since[^2]. |
| 1.4 | 2022 | Table- and column-level metadata. |
| 1.x | 2023 | JSS paper published (Bouchet-Valat & Kamiński)[^1]; precompilation improvements landing alongside Julia 1.9. |
| 1.x | 2026 | Actively maintained; last push 2026-07. |

## References

[^1]: Bouchet-Valat, M., & Kamiński, B. (2023). "DataFrames.jl: Flexible and Fast Tabular Data in Julia." Journal of Statistical Software, 107(4). https://doi.org/10.18637/jss.v107.i04
[^2]: DataFrames.jl release notes and 1.0 milestone. https://github.com/JuliaData/DataFrames.jl/releases
[^3]: Tables.jl — the generic table interface for Julia. https://github.com/JuliaData/Tables.jl
[^4]: Julia blog, "First-Class Statistical Missing Values Support in Julia 0.7" — 2018-06. https://julialang.org/blog/2018/06/missing/

## Tags

julia, dataframes, tabular-data, data-analysis, data-science, in-memory, split-apply-combine, statistics, columnar
