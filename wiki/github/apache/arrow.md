# apache/arrow

> A language-independent columnar memory format and cross-language toolbox for zero-copy data interchange and in-memory analytics.

[GitHub repo](https://github.com/apache/arrow) ·
[Official website](https://arrow.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/arrow/blob/main/LICENSE.txt)

## Overview

Apache Arrow is two things that are easy to conflate: a *specification* (the Arrow columnar memory format and its IPC/Flight wire protocols) and a *set of reference libraries* that implement it in more than a dozen languages. The specification is the durable artifact — it defines exactly how a table of typed, possibly-nested columns is laid out in memory so that two processes, written in different languages, can share the same bytes without serialization or copying[^1]. The libraries exist to produce and consume that layout.

The defining bet is **columnar-in-memory as a lingua franca**. Before Arrow, every system (pandas, Spark, a Parquet reader, a database driver) had its own in-memory representation, and moving data between them meant a serialize/deserialize round-trip that often dominated runtime. Arrow's value proposition is that if everyone agrees on one memory layout, that cost goes to zero: a Spark executor can hand a buffer to a Python UDF, or a Flight server can stream to a client, with no marshalling. The tradeoff is that Arrow is deliberately not a storage format, not a query engine, and not a dataframe library — it is the substrate those things are built on. Users frequently arrive expecting a product and find a foundation.

Arrow began in 2016, seeded by developers from the Impala, Spark, Pandas, and Drill communities, with Wes McKinney (pandas' creator) a central figure[^2]. It is an Apache Software Foundation project. Over time the monorepo has been progressively split: the Go, Java, JavaScript, .NET, Rust, Swift, and Julia implementations, plus ADBC, now live in separate repositories, while this repo (`apache/arrow`) is centered on the C++, Python (PyArrow), R, Ruby, C/GLib, and MATLAB implementations that share the C++ core.

## Getting Started

Python (PyArrow) is the most common entry point:

```bash
pip install pyarrow
```

```python
import pyarrow as pa
import pyarrow.parquet as pq

# Build a columnar table in memory (zero-copy from Python lists where possible)
table = pa.table({
    "id":   pa.array([1, 2, 3], type=pa.int64()),
    "name": pa.array(["tom", "brad", "anvil"]),
})

# Write and read Parquet without leaving the Arrow memory model
pq.write_table(table, "people.parquet", compression="zstd")
table2 = pq.read_table("people.parquet")

# Hand the same buffers to pandas with no serialization
df = table2.to_pandas()
```

C++ (the reference core) via conda or vcpkg; the Arrow website documents package feeds per platform[^3].

## Architecture / How It Works

The heart of the project is the **columnar format**: each column ("array") is stored as one or more contiguous buffers — a validity bitmap plus type-specific data/offset buffers. Fixed-width types are a single values buffer; variable-width types (strings, binary) add an offsets buffer; nested types (lists, structs, maps, unions) are composed recursively from child arrays. This layout is what makes vectorized, SIMD-friendly processing and zero-copy slicing possible.

Key components:

- **Arrow C++ core** — memory management (reference-counted off-heap `Buffer`s, memory pools), the array/table containers, and a `compute` kernel library. This is the code most other implementations in this repo bind to.
- **PyArrow** — a Cython binding over the C++ core, not a reimplementation. Its speed and its packaging weight both come from this. R and Ruby bindings similarly wrap C++ (R directly, Ruby via C/GLib).
- **IPC format** — the serialized form of Arrow data (streaming and file variants), using Google FlatBuffers for metadata so schema can be read without parsing the whole payload.
- **The Arrow C Data Interface** — a tiny, stable ABI (two C structs) that lets libraries pass Arrow arrays across language/library boundaries *without depending on Arrow itself*[^4]. This is how DuckDB, Polars, and others interoperate with Arrow while keeping their own memory managers.
- **Flight** — an RPC framework (gRPC + Arrow IPC) for high-throughput data services; Flight SQL layers a database-protocol vocabulary on top.
- **Gandiva** — an LLVM-based expression compiler that JIT-compiles projections/filters over Arrow buffers.
- **Parquet C++** — the Parquet reader/writer lives in this repo and is tightly coupled to the Arrow memory model; PyArrow is the de facto reference Parquet implementation for the Python ecosystem.

The most important architectural fact for consumers is the **C Data Interface**: it decouples "using Arrow-shaped data" from "depending on the Arrow libraries." Much of Arrow's real-world reach (Polars, DuckDB, pandas' Arrow-backed dtypes) flows through this ABI rather than through PyArrow itself.

## Production Notes

**PyArrow is heavy.** The wheel bundles the C++ core, Parquet, and often the full compute kernel set; installed size runs into the hundreds of megabytes and it is a frequent pain point for AWS Lambda size limits, slim Docker images, and cold-start latency. Teams that only need the memory format sometimes prefer `polars` or the C Data Interface to avoid pulling PyArrow.

**Format stability vs. library churn.** The columnar format is versioned conservatively and has strong backward-compatibility guarantees — data written years ago still reads. The libraries move faster: PyArrow has deprecated and removed APIs across major versions, and some functionality (the legacy `pyarrow.dataset` behaviors, `ParquetDataset` internals) has shifted. Pin PyArrow versions and read the changelog on upgrades.

**Type-mapping edge cases.** Arrow ↔ pandas/NumPy conversion is where most bugs live: nullability (Arrow has a validity bitmap; NumPy floats use NaN), timestamp timezones, nested/struct columns, and the distinction between `string` and large-`string`/`string_view` types. `to_pandas()` can also silently copy where you expected zero-copy if types don't align exactly.

**Memory accounting.** Arrow allocates off-heap. In a JVM (Java Arrow) or a Python process, RSS can be dominated by Arrow buffers invisible to the runtime's own heap tools; use Arrow's memory-pool statistics for accounting, and be aware that reference-counted buffers keep whole record batches alive if you retain a single slice.

**This is not a query engine.** The `compute` kernels handle element-wise and aggregate operations, and Arrow ships an experimental streaming execution engine (Acero), but for real query workloads people reach for DuckDB, Polars, or DataFusion (the last two build directly on Arrow). Choosing Arrow's own execution layer over those is usually the wrong call.

## When to Use / When Not

**Use when:**
- You move tabular data between systems or languages and serialization cost matters.
- You read/write Parquet, Feather, or CSV and want a fast, well-tested path (PyArrow is the reference).
- You are building a data tool and want interop with the broader ecosystem via a stable ABI (the C Data Interface).
- You need zero-copy sharing between a compute layer and, say, a Python UDF.

**Avoid (or look past it) when:**
- You want a dataframe API or a query engine — use Polars, pandas, DuckDB, or DataFusion, which sit *on* Arrow.
- Deployment size is tightly constrained (Lambda, edge) and you only need part of the functionality.
- Your data is small and single-process; the columnar/interop machinery buys you nothing over plain in-language structures.
- You need row-oriented, OLTP-style access patterns — columnar layout is the wrong shape.

## Alternatives

- pola-rs/polars — a dataframe/query engine built on Arrow memory; use it when you want an actual API, not a substrate.
- duckdb/duckdb — in-process analytical SQL engine that speaks Arrow zero-copy; use it when your problem is queries, not interchange.
- apache/parquet-format — on-disk columnar storage; use it for persistence, and use Arrow as the in-memory counterpart.
- protocolbuffers/protobuf — row-oriented cross-language serialization; use it for messages/RPC payloads, not columnar analytics.
- apache/arrow-rs — the Rust Arrow implementation (separate repo); use it when working in Rust or building on DataFusion.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2016-10 | First release; format still evolving[^2]. |
| 0.15.0 | 2019-10 | IPC format alignment change (RecordBatch body 8→64 byte), a known cross-version compatibility break. |
| 1.0.0 | 2020-07 | Format declared stable; forward/backward compatibility guarantees begin[^5]. |
| 4.0.0 | 2021-04 | Flight SQL groundwork, compute kernel expansion. |
| 7.0.0 | 2022-02 | Acero-era execution work, dataset improvements. |
| 12.0.0 | 2023-05 | Continued split of language implementations into separate repos. |
| 21.0.0 | 2025-07 | Recent major line; monorepo now C++/Python/R/Ruby-centric. |

(Arrow uses a single project-wide version number across the libraries in this repo; the format version evolves separately and far more slowly.)

## References

[^1]: Apache Arrow, "Arrow Columnar Format" specification. https://arrow.apache.org/docs/format/Columnar.html
[^2]: Wes McKinney, "Apache Arrow and the '10 Things I Hate About pandas'" — 2017-09-21. https://wesmckinney.com/blog/apache-arrow-pandas-internals/
[^3]: Apache Arrow, "Install" — package feeds per language/platform. https://arrow.apache.org/install/
[^4]: Apache Arrow, "The Arrow C Data Interface". https://arrow.apache.org/docs/format/CDataInterface.html
[^5]: Apache Arrow, "Format Versioning and Stability". https://arrow.apache.org/docs/format/Versioning.html

## Tags

c-plus-plus, python, columnar-format, in-memory-analytics, data-interchange, parquet, apache, zero-copy, dataframe, ipc, flight-rpc, big-data
