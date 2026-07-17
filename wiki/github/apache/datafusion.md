# apache/datafusion

> An extensible SQL query engine, shipped as a Rust library, for building your own analytic database on Apache Arrow.

[GitHub repo](https://github.com/apache/datafusion) ·
[Official website](https://datafusion.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/datafusion/blob/main/LICENSE.txt)

## Overview

DataFusion is a query engine written in Rust that uses Apache Arrow as its
in-memory columnar format. It is not a database — it is the set of parts you
assemble into one: a SQL parser and planner, a rule-based optimizer, a
vectorized multi-threaded execution engine, and pluggable data sources.
The project began as Andy Grove's personal Rust experiment, was donated to
Apache Arrow as a subproject, and was promoted to a top-level Apache Software
Foundation project in 2024 when the repository moved from `apache/arrow-datafusion`
to `apache/datafusion`[^1].

The defining tension is "toolkit, not product." DataFusion gives you a working
engine out of the box — `SELECT` over CSV/Parquet/JSON with predicate pushdown
and parallelism — but the intended audience is people building databases,
dataframe libraries, and custom query systems, not analysts running ad-hoc
queries. That reach shows in its user list: InfluxDB 3.0, GreptimeDB, LanceDB,
ParadeDB, Comet (a Spark accelerator), and dozens of other systems embed it
rather than reimplement a planner[^2]. If you want a finished analytical database
you reach for DuckDB; if you want to build one, you reach for DataFusion.

The cost of that generality is API churn. DataFusion iterates fast, bumps its
minimum Rust version regularly, and reshapes public traits between releases.
It publishes deprecation guidelines[^3], but embedders still budget for upgrade
work on nearly every version bump.

## Getting Started

DataFusion is a Rust crate; there is also a `datafusion-cli` binary for
interactive SQL.

```bash
# As a library dependency
cargo add datafusion tokio --features tokio/rt-multi-thread

# Or the standalone SQL CLI
cargo install datafusion-cli   # also available via Homebrew: brew install datafusion
```

```rust
use datafusion::prelude::*;

#[tokio::main]
async fn main() -> datafusion::error::Result<()> {
    let ctx = SessionContext::new();
    ctx.register_csv("example", "tests/data/example.csv", CsvReadOptions::new())
        .await?;

    let df = ctx
        .sql("SELECT a, MIN(b) FROM example WHERE a <= b GROUP BY a LIMIT 100")
        .await?;

    df.show().await?;   // executes and prints RecordBatches
    Ok(())
}
```

The same `DataFrame` handle is reachable through a fluent API
(`ctx.read_csv(...).filter(...).aggregate(...)`), and both the SQL and DataFrame
paths lower into the identical `LogicalPlan`.

## Architecture / How It Works

A query flows through distinct, individually replaceable stages:

1. **Parse** — SQL text is parsed by `sqlparser-rs` into an AST.
2. **Logical plan** — the AST becomes a `LogicalPlan` tree (`Projection`,
   `Filter`, `Aggregate`, `Join`, …) expressed over `Expr` nodes.
3. **Optimize** — a rule-based optimizer rewrites the logical plan: predicate
   pushdown, projection pushdown, constant folding, filter simplification,
   subquery decorrelation.
4. **Physical plan** — the logical plan is lowered to an `ExecutionPlan` tree,
   then optimized again (join-order heuristics, partition/repartition insertion,
   sort enforcement).
5. **Execute** — each `ExecutionPlan` produces a `SendableRecordBatchStream`:
   an async stream of Arrow `RecordBatch`es. Execution is partitioned across
   cores and vectorized against Arrow's compute kernels.

Execution is a pull-based streaming model — closer to vectorized Volcano than
to whole-stage codegen — where operators pull batches from their children on a
Tokio async runtime. Nearly every layer is an extension point: `TableProvider`
for custom sources, scalar/aggregate/window UDFs, custom `OptimizerRule`s,
custom physical operators, and a `CatalogProvider` for schema resolution. The
`SessionContext` and its `SessionState` carry configuration, registered
functions, and the catalog.

The codebase is split into many crates (`datafusion-common`, `datafusion-expr`,
`datafusion-physical-plan`, `datafusion-sql`, `datafusion-functions`, and more)
so embedders can depend on the planning layers without pulling the full engine.

## Production Notes

**It is a library, not a server.** There is no storage layer, no persistence, no
transactions, no wire protocol, no access control. Those are your job. Treating
DataFusion as a drop-in database is the most common misconception.

**Memory and spilling.** Query execution can OOM. DataFusion has a `MemoryPool`
abstraction and spill-to-disk for some memory-hungry operators (external sort,
grouped aggregation, some joins), but coverage is not universal — set memory
limits via the runtime config and test with representative data volumes rather
than assuming every operator spills gracefully.

**API churn is the real operational cost.** Public traits and method signatures
change between releases and the MSRV moves forward regularly. Systems that embed
DataFusion typically pin an exact version and schedule dedicated upgrade work.
This is the single most-cited pain among downstream projects.

**Compile times and binary size.** Pulling in Arrow plus the full DataFusion
crate produces long cold builds and large binaries. Trimming Cargo features
(`avro`, `compression`, `crypto_expressions`, etc.) and depending only on the
planning crates you need meaningfully cuts both.

**Performance is competitive but not universally best.** Parquet reading does
row-group pruning, page-index and predicate/projection pushdown, and the engine
parallelizes well. On single-node interactive analytics DuckDB and ClickHouse
often still win on specific workloads; DataFusion's advantage is customizability,
not topping every benchmark.

## When to Use / When Not

**Use when:**
- You are building a database, dataframe library, or custom query system in Rust
  and want a planner/optimizer/execution engine instead of writing one.
- You need SQL or a DataFrame API over Arrow, Parquet, CSV, JSON, or a custom
  `TableProvider`, with real optimization and multi-core execution.
- You need deep extensibility: custom functions, operators, optimizer rules, or
  data sources wired into a first-class plan.

**Avoid when:**
- You want a finished analytical database — use DuckDB or ClickHouse.
- You are not working in Rust and don't want to embed via the Python/Java
  bindings (those are separate repos with their own release cadence).
- You need a frozen, rarely-breaking API; DataFusion iterates quickly.

## Alternatives

- duckdb/duckdb — embeddable analytical database; use when you want a complete product, not a toolkit to build one.
- pola-rs/polars — Arrow-based DataFrame library; use when you want fast DataFrame analytics in Rust/Python without assembling an engine.
- apache/arrow-rs — the Rust Arrow implementation DataFusion is built on; use directly when you only need columnar arrays and compute kernels.
- ClickHouse/ClickHouse — production OLAP database server; use when you want a deployable columnar warehouse, not a library.
- facebookincubator/velox — C++ execution-engine toolkit in the same "build-your-own-engine" niche; use when your host system is C++.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2018 | Andy Grove starts DataFusion as a Rust SQL engine[^1]. |
| — | 2019 | Donated to the Apache Arrow project as a subproject. |
| — | 2021-04 | Split into its own `apache/arrow-datafusion` repository[^4]. |
| — | 2024 | Promoted to a top-level Apache Software Foundation project; repo renamed to `apache/datafusion`[^1]. |

DataFusion uses frequent, sequentially numbered releases (40+ major versions by
2026); consult the changelog for exact per-version dates rather than relying on
memory[^5].

## References

[^1]: "Apache DataFusion Graduates to Top-Level Apache Project." Apache Arrow blog / DataFusion announcements. https://arrow.apache.org/blog/
[^2]: DataFusion "Known Users." https://datafusion.apache.org/user-guide/introduction.html#known-users
[^3]: DataFusion API health and deprecation guidelines. https://datafusion.apache.org/contributor-guide/api-health.html
[^4]: apache/datafusion repository, created 2021-04-17. https://github.com/apache/datafusion
[^5]: DataFusion changelog. https://github.com/apache/datafusion/tree/main/dev/changelog

## Tags

rust, sql, query-engine, apache-arrow, olap, dataframe, columnar, parquet, analytics, big-data, embedded-database
