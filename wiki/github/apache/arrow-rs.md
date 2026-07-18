# apache/arrow-rs

> The official Rust implementation of Apache Arrow (columnar in-memory format) and Apache Parquet (columnar file format) — the substrate under most of the Rust data ecosystem.

[GitHub repo](https://github.com/apache/arrow-rs) ·
[Official website](https://arrow.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/arrow-rs/blob/main/LICENSE.txt)

## Overview

arrow-rs is the Apache Software Foundation's native Rust implementation of the Arrow columnar specification[^1], bundled in one repository with a full-featured Parquet reader/writer and an Arrow Flight (gRPC) implementation. The Rust implementation originally lived inside the `apache/arrow` monorepo and was split into this standalone repository in April 2021[^2]; since then it has versioned and released independently of the C++/Java Arrow releases, which is why its major version numbers (59.x as of mid-2026) bear no relation to the main Arrow project's.

Its 3.5k GitHub stars badly undercount its importance: this is infrastructure, not an application. DataFusion, InfluxDB 3, delta-rs, Lance/LanceDB, GreptimeDB, and most other Rust analytics projects sit directly on these crates, and `arrow` and `parquet` are among the most-downloaded data crates on crates.io. Development is correspondingly active — monthly releases, pushes to `main` most days, and ~600 open issues that reflect scale rather than neglect.

The defining tension is **release velocity vs. ecosystem stability**. For years arrow-rs shipped a breaking major release roughly every other month; because Arrow types (`RecordBatch`, `ArrayRef`) appear in downstream public APIs, every major bump forced a coordinated upgrade wave across the ecosystem. Since 2024 the project has throttled to at most one breaking major per quarter with minor releases in between[^3] — an explicit acknowledgment that the churn was the project's biggest cost.

## Getting Started

```bash
cargo add arrow parquet
```

Build a `RecordBatch`, round-trip it through Parquet:

```rust
use std::{fs::File, sync::Arc};
use arrow::array::{Int32Array, StringArray};
use arrow::datatypes::{DataType, Field, Schema};
use arrow::record_batch::RecordBatch;
use parquet::arrow::arrow_writer::ArrowWriter;
use parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let schema = Arc::new(Schema::new(vec![
        Field::new("id", DataType::Int32, false),
        Field::new("name", DataType::Utf8, false),
    ]));
    let batch = RecordBatch::try_new(schema.clone(), vec![
        Arc::new(Int32Array::from(vec![1, 2, 3])),
        Arc::new(StringArray::from(vec!["a", "b", "c"])),
    ])?;

    let mut writer = ArrowWriter::try_new(File::create("data.parquet")?, schema, None)?;
    writer.write(&batch)?;
    writer.close()?;

    let reader = ParquetRecordBatchReaderBuilder::try_new(File::open("data.parquet")?)?.build()?;
    for batch in reader {
        println!("{} rows", batch?.num_rows());
    }
    Ok(())
}
```

## Architecture / How It Works

The `arrow` crate is an umbrella over a family of subcrates (`arrow-array`, `arrow-buffer`, `arrow-schema`, `arrow-data`, `arrow-cast`, `arrow-ipc`, `arrow-select`, `arrow-ord`, and others), a split performed during 2022 primarily to cut compile times — depend on the narrow subcrates directly if build time matters.

Core memory model, bottom up:

- **`Buffer`** — a reference-counted, immutable, aligned byte region. Slicing is zero-copy: a slice is an offset+length view holding a refcount on the parent allocation.
- **`ArrayData`** — buffers + optional validity (null) bitmap + child `ArrayData` for nested types, plus a `DataType`. This is the untyped spec-level representation.
- **Typed arrays** — `PrimitiveArray<T>`, `StringArray` (i32 offsets), `LargeStringArray` (i64), `ListArray`, `StructArray`, `DictionaryArray`, and since ~2024 the view types (`StringViewArray`/`BinaryViewArray`) that store short strings inline and long ones by pointer, which DataFusion adopted for significant string-heavy query wins[^4].
- **`RecordBatch`** — a schema plus same-length columns; the unit of exchange.

Compute kernels (cast, comparison, arithmetic, sort, filter, take) live in the subcrates and lean on LLVM auto-vectorization rather than hand-written SIMD. `arrow-row` provides a row-oriented comparable encoding used for multi-column sorts and joins. The C Data Interface (FFI) enables zero-copy exchange with pyarrow and other implementations. The `parquet` crate supports sync and async (Tokio) readers, predicate pushdown via row filters, page-index pruning, and bloom filters, and is generally regarded as one of the fastest Parquet implementations available.

## Production Notes

**Version lockstep is the classic footgun.** `RecordBatch` from arrow 58 and arrow 59 are distinct Rust types. If two of your dependencies (say datafusion and delta-rs) pin different arrow majors, they cannot exchange data without an FFI round-trip. Upgrades therefore ripple: you wait for every arrow-consuming dependency to publish against the new major before you can move. Pin exact majors and upgrade the whole tree at once. The quarterly-major schedule[^3] and the "deprecate for two majors before removal" policy soften but do not remove this.

**Compile times.** The umbrella `arrow` crate with default features pulls in CSV, JSON, and IPC support you may not need. Depending on `arrow-array` + `arrow-buffer` + `arrow-schema` directly, and trimming `parquet` features, is a meaningful build-time win.

**Offset limits.** `StringArray`/`BinaryArray`/`ListArray` use i32 offsets — a single array's variable-length data is capped at 2 GiB. Overflow panics at construction. Use `Large*` variants (i64) or view arrays for large payloads, and remember most kernels require both sides to agree on the offset width.

**Panics vs. errors.** Project policy is `Result` for invalid user input, panics for unreachable states[^5] — but the boundary is judgment-based and some kernel paths panic on inputs you might expect to be recoverable (e.g. length mismatches, overflow in non-checked arithmetic kernels). Fuzz untrusted-input paths; treat IPC/Parquet decoding of untrusted bytes with care.

**Zero-copy slices retain parent memory.** Slicing a large batch and keeping a small slice alive keeps the whole underlying buffer allocated — a common source of surprise memory retention in long-running services. Copy (`concat` or `take`) to detach.

**`object_store` moved out.** The generic object-storage crate that lived here for years now resides in `apache/arrow-rs-object-store`[^6]; old docs and blog posts pointing at this repo are stale.

**MSRV** is rolling, bumped only in major releases, and guaranteed at least six months old — comfortable for most, relevant for slow-moving corporate toolchains.

## When to Use / When Not

**Use when:**
- You are building analytics infrastructure in Rust — query engines, table formats, time-series stores — and need the ecosystem-standard columnar type.
- You need a fast, feature-complete Parquet reader/writer (async, pushdown, page index) in a non-JVM language.
- You need zero-copy interchange with pyarrow, DuckDB, or anything else speaking the Arrow C Data Interface or Flight.

**Avoid when:**
- You want a DataFrame API for end-user data wrangling — this is a format library; use Polars or DataFusion on top instead.
- Your data is row-oriented OLTP-shaped (point lookups, frequent single-row mutation); Arrow arrays are immutable and columnar, and mutation means rebuild.
- You cannot absorb quarterly breaking majors in a dependency that leaks into your public API.

## Alternatives

- pola-rs/polars — vendors its own Arrow fork (`polars-arrow`) behind a DataFrame API; use it when you want dataframes, not a format library.
- apache/arrow — the C++/Python/Java monorepo; use pyarrow when your host language is Python.
- apache/arrow-nanoarrow — minimal C implementation; use it for embedding Arrow ABI support in small libraries without a heavy dependency.
- apache/datafusion — not a competitor but the layer above: use it when you want SQL/DataFrame execution rather than hand-written kernels.
- jorgecarleitao/arrow2 — historical safety-focused rewrite, now unmaintained; the community reconverged on arrow-rs. Do not start new projects on it.

## History

| Version | Date | Notes |
|---------|------|-------|
| (monorepo) | 2018–2021 | Rust implementation developed inside apache/arrow, versioned with the main project. |
| 4.0.0 | 2021-04 | Repository split; first standalone releases, independent versioning begins[^2]. |
| ~20–30 | 2022 | `arrow` progressively split into `arrow-*` subcrates; bi-monthly breaking majors. |
| 50.0.0 | 2024-01 | Release-cadence rework begins (#5368): majors at most quarterly, minors monthly[^3]. |
| 5x | 2024–2025 | View arrays mature and see downstream adoption[^4]; `object_store` moved to its own repository[^6]. |
| 59.x | 2026 | Current major line; 60.0.0 (breaking) scheduled for 2026-08[^3]. |

## References

[^1]: Apache Arrow Columnar Format specification. https://arrow.apache.org/docs/format/Columnar.html
[^2]: apache/arrow-rs repository (created 2021-04-17); release history at https://crates.io/crates/arrow/versions
[^3]: arrow-rs release schedule discussion, issue #5368. https://github.com/apache/arrow-rs/issues/5368
[^4]: Apache Arrow blog, "Using StringView / German-style strings in DataFusion" (2024). https://arrow.apache.org/blog/
[^5]: arrow-rs panic vs. `Result` guidelines, issue #6737. https://github.com/apache/arrow-rs/issues/6737
[^6]: arrow-rs-object-store repository. https://github.com/apache/arrow-rs-object-store

## Tags

rust, apache-arrow, parquet, columnar-format, in-memory-analytics, data-engineering, zero-copy, serialization, grpc, library
