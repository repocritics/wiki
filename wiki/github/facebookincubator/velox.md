# facebookincubator/velox

> A composable C++ execution engine library for building data systems — the
> shared "muscle" beneath Presto, Spark-via-Gluten, and Meta's internal engines.

[GitHub repo](https://github.com/facebookincubator/velox) ·
[Official website](https://velox-lib.io/) ·
[License: Apache-2.0](https://github.com/facebookincubator/velox/blob/main/LICENSE)

## Overview

Velox is not a database. It is a C++ library of vectorized execution
components — types, columnar vectors, expression evaluation, relational
operators, file-format readers, storage connectors, memory management — that
engine developers assemble into query engines. It deliberately ships **no SQL
parser, no optimizer, and no dataframe API**: it takes a fully optimized query
plan as input and executes it[^1]. Meta created it to consolidate the dozens
of divergent execution runtimes inside the company (Presto, Spark, streaming,
ML preprocessing) into one reusable engine; development went public on GitHub
in 2021, the announcement and VLDB paper followed in 2022[^1][^2], and it is
now co-developed with IBM, Intel, Voltron Data, Microsoft, ByteDance and
others.

The star count (~4.2k) understates its footprint: end users touch Velox only
through **Prestissimo** (Presto's C++ worker, the engine under IBM
watsonx.data), **Apache Gluten (incubating)** (which swaps Spark's JVM
execution for Velox)[^3], and Meta's internal fleet. The defining tension is
unification versus churn: Velox promises "one engine, correct everywhere
Presto and Spark disagree," but delivers it via a live-at-head model with no
versioned releases, pushing real integration cost onto downstream consumers.

## Getting Started

Velox is built from source; there is no package-manager distribution.

```bash
git clone https://github.com/facebookincubator/velox.git
cd velox
./scripts/setup-ubuntu.sh   # or setup-macos.sh / setup-centos9.sh
make                        # make debug | make release | make unittest
```

Minimal use — register functions, build a plan, hand it to a
`velox::exec::Task` (sketch; see `velox/examples/` for runnable programs):

```cpp
#include "velox/exec/tests/utils/PlanBuilder.h"
#include "velox/functions/prestosql/registration/RegistrationFunctions.h"
using namespace facebook::velox;

functions::prestosql::registerAllScalarFunctions();
auto plan = exec::test::PlanBuilder()
                .tableScan(ROW({"a", "b"}, {BIGINT(), BIGINT()}))
                .filter("a > 0")
                .project({"a + b AS s"})
                .planNode();
```

## Architecture / How It Works

Velox is organized as layered, individually reusable components[^1]:

- **Type & Vector** — an Arrow-compatible columnar layout with Flat,
  Dictionary, Constant, and RLE encodings, lazy materialization, and
  out-of-order writes. Velox and Arrow converged on formerly incompatible
  layouts (notably the string-view representation Arrow later adopted), making
  zero-copy interchange with Arrow-based systems the common path.
- **Expression evaluation** — fully vectorized, with dictionary peeling,
  common-subexpression reuse, and constant folding. Correctness is enforced by
  fuzzers that cross-check results against Presto/Spark reference semantics.
- **Functions** — parallel packages implementing **Presto** and **Spark SQL**
  semantics, because the engines genuinely disagree (null handling, casts,
  decimals, timestamps). A host engine registers the package for its dialect.
- **Operators & Drivers** — relational operators (scan, project, hash/merge/
  nested-loop joins, aggregation, ordering, exchange, unnest) composed into
  per-Task pipelines executed by Driver threads; parallelism comes from
  multiple Drivers per pipeline and repartitioning via exchanges.
- **I/O & connectors** — pluggable sources supporting ORC/DWRF, Parquet, and
  Nimble (Meta's newer columnar format, open-sourced 2024) over S3, HDFS,
  GCS, ABFS, or local files, with async prefetch and an SSD-backed cache.
- **Serializers** — wire formats for shuffles: PrestoPage and Spark UnsafeRow.
- **Resource management** — memory pools, arenas, spilling, and task
  scheduling, designed to integrate with a host engine's memory arbitration.

Every layer has an extension point: custom types, functions, operators, file
formats, storage adapters, serializers. That extensibility is the product —
Velox competes on being embeddable, not complete.

## Production Notes

**There are no releases.** Velox does not publish semver versions or stable
API/ABI guarantees; consumers (Gluten, Prestissimo) pin git commits and absorb
breaking changes continuously — the single largest adoption cost.

**The build is heavy, the hardware floor real.** Large dependency tree (folly,
boost, fmt, xsimd, ICU, protobuf, ...) managed by per-platform setup scripts;
full builds take substantial time and RAM — the README documents
`BUILD_THREADS` specifically to avoid OOM kills during parallel linking (a
docker-compose path exists). Requires GCC 11+/Clang 15+; x86 CPUs must support
BMI/BMI2/F16C, with AVX/AVX2 used where available; ARM uses Neon. Older or
exotic CPUs are out of scope.

**Semantic coverage is incomplete.** Presto/Spark function and operator
coverage is broad but not total. Gluten handles gaps by falling back to
vanilla Spark JVM execution per-operator — each fallback boundary pays
columnar-to-row conversion and can erase the speedup, so profile which of
*your* queries stay on the Velox path.

**Memory integration is nontrivial.** Embedding Velox means reconciling two
memory accounting systems; Prestissimo and Gluten both carry significant
arbitration/OOM glue code — plan for the same.

## When to Use / When Not

**Use when:**
- You are building or accelerating a query engine and want vectorized C++
  execution without writing operators, expression eval, and format readers.
- You run Spark at scale and want CPU savings via Gluten, or Presto via
  Prestissimo, with an existing community carrying the integration.
- You need Presto- or Spark-exact semantics in native code.

**Avoid when:**
- You want a working analytical database — Velox has no parser or optimizer;
  DuckDB or ClickHouse give you a complete system today.
- You cannot staff continuous tracking of an unversioned C++ dependency.
- Your workload is small enough that JVM Spark or a managed warehouse is
  fine; the integration cost only pays off at scale.

## Alternatives

- apache/datafusion — embeddable Rust engine *with* SQL parser and optimizer;
  use it when you want a full engine toolkit and Rust over C++.
- duckdb/duckdb — complete in-process analytical database; use it when you
  want answers, not components.
- ClickHouse/ClickHouse — standalone OLAP server; use it when you need a
  deployable system rather than a library.
- apache/arrow — Acero, Arrow C++'s compute engine; lighter, less complete
  operator surface, already in-tree if you depend on Arrow.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2021-07 | Public development on GitHub under facebookincubator. |
| Public launch | 2022-08 | Meta engineering blog + VLDB 2022 paper[^1][^2]. |
| IBM acquires Ahana | 2023-04 | Major backer; Velox powers Presto C++ in watsonx.data. |
| Gluten enters Apache incubation | 2024-01 | Spark-on-Velox goes community-governed[^3]. |
| Nimble open-sourced | 2024-04 | Meta's next-gen columnar file format joins the stack. |
| OpenZL in Nimble OSS | 2026-07 | Compression work continues; ~weekly technical blog cadence[^4]. |

## References

[^1]: Pedreira et al., "Velox: Meta's Unified Execution Engine," VLDB 2022. https://www.vldb.org/pvldb/vol15/p3372-pedreira.pdf
[^2]: Meta Engineering, "Velox: Meta's unified execution engine" — 2022. https://engineering.fb.com/2022/08/31/open-source/velox/
[^3]: Apache Gluten (incubating) — Spark plugin offloading execution to native engines including Velox. https://gluten.apache.org/
[^4]: Velox blog, "Making OpenZL Available in Nimble OSS" — 2026-07-05. https://velox-lib.io/blog/openzl-in-nimble-oss

## Tags

cpp, query-execution, vectorized-execution, columnar, data-management, database-internals, arrow, presto, spark, analytics, meta
