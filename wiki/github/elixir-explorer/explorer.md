# elixir-explorer/explorer

> dplyr-flavored series and dataframes for Elixir, executed by Polars through Rust NIFs.

[GitHub repo](https://github.com/elixir-explorer/explorer) ·
[Official docs](https://hexdocs.pm/explorer) ·
[License: MIT](https://github.com/elixir-explorer/explorer/blob/main/LICENSE)

## Overview

Explorer is the de facto dataframe library of the Elixir ecosystem: series (one-dimensional, single-dtype) and dataframes (two-dimensional) with an API modeled on R's dplyr and the Tidy Data conventions rather than on pandas[^1]. It was started in 2021 by Chris Grainger (Amplified) and is co-maintained with José Valim; the repo began life in the `elixir-nx` organization and later moved to its own `elixir-explorer` org[^2]. At ~1.3k stars it is small by GitHub-wide standards but central within Elixir — it is the tabular-data layer of the Nx/Livebook stack, and v0.12.0 shipped in July 2026, so it is actively maintained.

The defining design decision is that Explorer is a **binding, not an engine**. All computation happens in the Rust Polars library, called through NIFs; the Elixir layer contributes a deliberately constrained, immutable, "verbs"-style API ("by constraining your options, it helps you think")[^1]. The tradeoff cuts both ways: you get Polars-class performance from a language with no native numerics story, but you inherit native-code operational concerns inside the BEAM, a pinned Rust nightly toolchain for source builds, and an API surface that intentionally lags what raw Polars exposes.

Dtypes are simply typed and nullable: integers of 8–64 bits, 32/64-bit floats, `:string`, `:binary`, `:boolean`, `:category`, `:date`, `:time`, `:datetime`, `:duration`, `:list`, and `:struct`[^1]. IO covers CSV, Parquet, NDJSON, and Arrow IPC, plus S3-style object storage and databases via the companion ADBC bindings[^3]. Backends are pluggable in principle; in practice Polars is the first and only production backend.

## Getting Started

```elixir
Mix.install([{:explorer, "~> 0.12.0"}])
```

```elixir
require Explorer.DataFrame, as: DF

mountains =
  DF.new(
    name: ["Everest", "K2", "Aconcagua"],
    elevation: [8848, 8611, 6962]
  )

# Query macros: refer to columns bare, call aggregations inline
DF.filter(mountains, elevation > mean(elevation))
#=> Polars[2 x 2]  name: ["Everest", "K2"]
```

The `require` is not boilerplate — `Explorer.DataFrame` exposes macros (`filter/2`, `mutate/2`, `summarise/2`, ...) that compile the query DSL. A precompiled NIF binary is downloaded at install time for common targets; set `EXPLORER_BUILD=1` and add `:rustler` to force a local build[^1]. The "Ten Minutes to Explorer" Livebook is the canonical tour[^4].

## Architecture / How It Works

The stack is three layers. On top, `Explorer.Series` / `Explorer.DataFrame` define the public API and enforce immutability — every operation returns a new series or frame, never mutating in place, even where mutation would be cheaper. In the middle, `Explorer.Query` translates the macro DSL (`elevation > mean(elevation)`) into backend expression structs at compile time, which is why column references work without quoting or anonymous functions. At the bottom, a Rustler-generated NIF crate wraps Polars; dataframes held from Elixir are NIF resource handles pointing at Arrow-backed memory owned by Rust.

Consequences of that shape:

- **Memory lives outside the BEAM.** Frame data sits on the Rust/Arrow heap, not the Erlang heap. Immutable-looking operations are cheaper than they appear because Polars shares Arrow buffers under the hood, but the data is invisible to `:erlang.memory/0` and to most BEAM observability tooling.
- **Compute bypasses the scheduler model.** Heavy queries execute in native code (on dirty schedulers), not as preemptible Elixir reductions. One big `group_by` does not yield the way idiomatic Elixir code does.
- **Distribution is explicit.** Since v0.9, series and dataframes can be placed on and transferred between Erlang nodes as remote references, keeping data next to computation in a cluster instead of copying frames around[^5].
- **Precompilation is the install path.** `RustlerPrecompiled` ships binaries for ~9 targets (macOS/Linux/Windows/FreeBSD, x86-64 and aarch64); source builds require a pinned Rust *nightly* and possibly CMake[^6].

## Production Notes

- **NIF crash = VM crash.** A panic or memory-safety bug in the native layer takes down the whole BEAM, bypassing supervision trees entirely. Polars and the binding are mature, but this is a categorically different failure mode from pure-Elixir libraries, and worth weighing for long-lived systems that lean on "let it crash" isolation.
- **"Illegal instruction" on older CPUs.** Default precompiled artifacts enable modern CPU features; on older x86-64 hardware the fix is `config :explorer, use_legacy_artifacts: true` at compile time[^1]. This surfaces as a process-killing signal, not an Elixir exception.
- **Precompiled binaries only exist for Hex releases.** Depending on a git ref of Explorer means compiling Polars yourself — with the pinned nightly toolchain — which can take a long time in CI. Pin Hex versions.
- **Feature gaps by target.** NDJSON and S3 IO are disabled on RISC-V because upstream dependencies do not compile there[^1].
- **API lags Polars.** The constrained API is a feature, but when you need a Polars capability Explorer has not wrapped, there is no escape hatch to raw expressions — you wait, PR it, or restructure the query.
- **Pre-1.0 churn.** Five years in, versioning is still 0.x with breaking changes landing in minor releases (dtype notation and API renames have shifted across versions). Read the changelog before every bump[^7].
- **Elixir-side iteration is a smell.** Converting frames to lists to map over rows discards the performance model. Keep work inside the query DSL; use `Series.transform/2` sparingly and expect it to be slow.

## When to Use / When Not

**Use when:**

- You are already on Elixir/Phoenix and need real tabular analytics without shelling out to a Python service.
- You work in Livebook — Explorer is its native dataframe layer and the notebooks integration is first-class.
- Your workload fits the dplyr verb set: filter, mutate, group, summarise, join over CSV/Parquet/Arrow data.
- You want Polars-level throughput with an immutable, pipeline-friendly API.

**Avoid when:**

- Your team's data work lives in Python — pandas/Polars there have a far larger ecosystem (plotting, ML, IO connectors) than Elixir will offer.
- You need Polars features Explorer has not wrapped, or maximum control over the query plan — use Polars directly.
- You cannot tolerate native-code risk in the BEAM, or cannot use precompiled binaries and cannot afford nightly-Rust source builds.
- Your "dataframes" are really OLAP SQL — a database (or DuckDB) is a better home than in-process frames.

## Alternatives

- pola-rs/polars — use the engine directly (Python/Rust) when you are not committed to Elixir; strictly more features, same core performance.
- elixir-dux/dux — DuckDB-backed Elixir dataframe API; prefer it when your workload is SQL-shaped or exceeds memory[^1].
- pandas-dev/pandas — the Python default; larger ecosystem, mutable API, slower engine.
- elixir-nx/nx — Elixir tensors for numerics/ML; complementary rather than competing — reach for it when data is numeric arrays, not tables.
- duckdb/duckdb — embedded analytical SQL; better than any dataframe library for larger-than-memory joins and aggregations.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2022-04-26 | First Hex release; Polars backend via Rustler[^7]. |
| v0.5.0 | 2023-01-12 | Rapid 0.5.x patch series through early 2023[^7]. |
| v0.7.0 | 2023-08-28 | Continued dtype and query expansion[^7]. |
| v0.9.0 | 2024-07-26 | Remote series/dataframes across nodes[^5]. |
| v0.10.0 | 2024-10-24 | Polars upgrades, API refinements[^7]. |
| v0.11.0 | 2025-07-12 | Release cadence slows to roughly yearly[^7]. |
| v0.12.0 | 2026-07-08 | Current release; still pre-1.0[^7]. |

## References

[^1]: Explorer README / hexdocs — features, dtypes, precompilation, legacy artifacts, Dux pointer. https://hexdocs.pm/explorer
[^2]: GitHub repository (created 2021-07-23; CI badges still reference the original elixir-nx org). https://github.com/elixir-explorer/explorer
[^3]: ADBC — Arrow Database Connectivity bindings for Elixir. https://github.com/elixir-explorer/adbc
[^4]: "Ten Minutes to Explorer" guide. https://hexdocs.pm/explorer/exploring_explorer.html
[^5]: Explorer v0.9 remote dataframes/series. https://github.com/elixir-explorer/explorer/blob/main/CHANGELOG.md
[^6]: RustlerPrecompiled. https://hexdocs.pm/rustler_precompiled
[^7]: Explorer changelog and releases. https://github.com/elixir-explorer/explorer/releases

## Tags

elixir, dataframes, data-science, polars, rust, nif, data-exploration, tidy-data, livebook, beam
