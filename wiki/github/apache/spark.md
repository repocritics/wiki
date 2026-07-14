# apache/spark

> A unified analytics engine for large-scale data processing — the JVM cluster engine that made SQL, streaming, and ML share one execution model.

[GitHub repo](https://github.com/apache/spark) ·
[Official website](https://spark.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/spark/blob/master/LICENSE)

## Overview

Apache Spark is a distributed data-processing engine that started at UC Berkeley's AMPLab (Matei Zaharia) around 2009, was open-sourced in 2010, and became a top-level Apache project in 2014[^1]. Its founding idea was the Resilient Distributed Dataset (RDD): an immutable, partitioned collection whose lineage is recorded so lost partitions can be recomputed instead of replicated[^2]. That let Spark keep working sets in memory across iterations, which is where it beat Hadoop MapReduce on iterative and interactive workloads.

Nobody writes RDDs by hand anymore. The center of gravity moved to the DataFrame / Dataset API and Spark SQL, where user code becomes a logical plan that the Catalyst optimizer rewrites and the Tungsten engine compiles to JVM bytecode. On top of that one engine sit four higher-level libraries — Spark SQL, Structured Streaming, MLlib, and GraphX — plus the pandas API on Spark. The pitch is genuinely "one engine, many workloads": batch, SQL, streaming, and ML share the same scheduler, shuffle, and optimizer.

The defining tension is that Spark is a heavy, JVM-based, cluster-oriented system in an era where a lot of "big data" fits on one large machine. It carries real operational weight — a driver, executors, a cluster manager, shuffle, JVM GC tuning — and for datasets under roughly a hundred gigabytes, single-node tools (DuckDB, Polars) now often win on both speed and simplicity. Spark's continued relevance is at the scale where data genuinely does not fit on one box, and in the enormous installed base of existing pipelines.

## Getting Started

```bash
pip install pyspark        # Python; pulls the bundled JVM assembly
# or download a prebuilt tarball from https://spark.apache.org/downloads.html
```

```python
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName("example").getOrCreate()

df = spark.read.parquet("s3://bucket/events/")
top = (
    df.filter(F.col("status") == "purchase")
      .groupBy("country")
      .agg(F.sum("amount").alias("revenue"))
      .orderBy(F.desc("revenue"))
)
top.show()          # nothing ran until this action triggered execution
```

The Scala shell (`./bin/spark-shell`) and `./bin/pyspark` give an interactive session with a pre-built `spark` context. `spark.range(1000 * 1000 * 1000).count()` is the canonical smoke test.

## Architecture / How It Works

A Spark application is a **driver** process (runs your code, holds the `SparkSession`, schedules work) coordinating a set of **executor** JVMs (run tasks, hold cached data). A **cluster manager** — Standalone, YARN, or Kubernetes — allocates the executors. Apache Mesos support was deprecated and removed.

Execution is **lazy**. Transformations (`filter`, `groupBy`, `join`) build a logical plan; only an **action** (`count`, `write`, `collect`) triggers a job. The DAG scheduler splits the job at shuffle boundaries into **stages**, each stage into **tasks** (one per partition), and dispatches tasks to executors.

The DataFrame/SQL path runs through two components:
- **Catalyst** — a rule-based + cost-based optimizer that rewrites the logical plan (predicate pushdown, column pruning, join reordering) and picks physical operators[^3].
- **Tungsten** — the execution engine: off-heap binary memory layout, cache-aware data structures, and whole-stage code generation that collapses a chain of operators into a single generated Java method (Spark 2.0)[^4].

**Adaptive Query Execution (AQE)**, on by default since 3.2, re-optimizes plans mid-flight using runtime statistics — coalescing shuffle partitions, switching join strategies, and splitting skewed partitions[^5]. It is one of the biggest practical wins of the 3.x line.

**Spark Connect** (introduced 3.4, stabilized in 4.0) decouples the client from the driver: the client sends an unresolved logical plan over gRPC to a remote Spark server[^6]. This is how PySpark is moving off its historical Py4J bridge, and it lets thin clients (notebooks, IDEs, other languages) talk to a shared cluster without co-locating a JVM.

The whole stack is Scala on the JVM. PySpark and SparkR are language bindings; the SparkR/R API is deprecated as of the 4.x line.

## Production Notes

**Shuffle is the cost center.** Any `groupBy`, `join`, or repartition that crosses partition boundaries writes intermediate data to disk and pulls it across the network. Most Spark tuning is really shuffle tuning: `spark.sql.shuffle.partitions` (the default 200 is wrong for both tiny and huge jobs), broadcast joins for small tables (`spark.sql.autoBroadcastJoinThreshold`), and avoiding wide transformations where possible.

**Data skew is the classic outage.** One key with disproportionately many rows sends one task orders of magnitude more work; the whole stage waits on that straggler, often ending in executor OOM. AQE's skew handling mitigates the common cases; salting keys is the manual fix.

**JVM memory and GC.** Executors split memory between execution (shuffle/sort/aggregation) and storage (cached data). Under-provisioned executors spill to disk or die with `OutOfMemoryError`; over-large heaps cause long GC pauses. This tuning is unavoidable and is the single most common source of Spark operational pain.

**PySpark UDF overhead.** A plain Python UDF serializes each row across the JVM↔Python boundary one at a time — often 10–100× slower than an equivalent built-in expression. Use native `pyspark.sql.functions`, and when you must write Python logic, use Arrow-based pandas UDFs (`applyInPandas` / vectorized UDFs) to batch the crossing.

**The small-files problem.** Writing many tiny output files (common with streaming or over-partitioned writes) crushes downstream read performance and metastore listing. Coalesce/repartition before writing, or compact after.

**Upgrade pains.** Spark upgrades are cross-cut by the Scala version (2.11 → 2.12 → 2.13) and the JVM (Java 8 → 11 → 17 → 21); Spark 4.x requires Java 17+. Hadoop client versions must match your storage cluster. Spark 4.0 made **ANSI SQL mode the default**, which turns previously-silent behaviors (overflow, invalid casts returning null) into runtime errors — a correctness improvement that will surface latent bugs on upgrade[^7].

## When to Use / When Not

**Use when:**
- Your data genuinely does not fit on a single machine and needs horizontal scale.
- You want one engine spanning batch ETL, SQL, streaming, and ML rather than stitching separate systems.
- You already run a Spark/lakehouse platform (Databricks, EMR, Dataproc, or self-hosted on Kubernetes/YARN).
- You need fault-tolerant recomputation over unreliable commodity clusters.

**Avoid when:**
- Your dataset fits comfortably on one large node — DuckDB or Polars will be faster with far less operational overhead.
- You need low-latency (sub-second, per-event) streaming — Structured Streaming is micro-batch-oriented; Flink is a better fit.
- Your workload is small and you cannot justify a cluster, a driver, and JVM tuning.
- You want interactive federated queries across many sources without moving data — a query engine like Trino fits better.

## Alternatives

- duckdb/duckdb — use instead when your analytics fit on one machine and you want an in-process OLAP engine with no cluster.
- pola-rs/polars — use instead for fast single-node dataframe work in Rust/Python without the JVM.
- apache/flink — use instead when you need true low-latency, event-at-a-time stream processing.
- dask/dask — use instead when you want Python-native distributed computing that stays close to NumPy/pandas semantics.
- trinodb/trino — use instead for interactive, federated SQL across heterogeneous sources rather than a full processing engine.
- ray-project/ray — use instead when the primary workload is distributed Python ML/training rather than SQL/ETL.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2010 | Open-sourced from UC Berkeley AMPLab; RDD model[^2]. |
| — | 2014-02 | Became an Apache top-level project[^1]. |
| 1.0 | 2014-05 | First stable release; Spark SQL introduced. |
| 1.3 | 2015-03 | DataFrame API + Catalyst optimizer. |
| 2.0 | 2016-07 | Dataset/DataFrame unified, Structured Streaming, whole-stage codegen[^4]. |
| 2.3 | 2018-02 | Native Kubernetes scheduler backend. |
| 3.0 | 2020-06 | Adaptive Query Execution, dynamic partition pruning[^5]. |
| 3.2 | 2021-10 | AQE on by default; pandas API on Spark (Koalas merged). |
| 3.4 | 2023-04 | Spark Connect introduced (gRPC client/server)[^6]. |
| 3.5 | 2023-09 | Last major 3.x line; Spark Connect maturing. |
| 4.0 | 2025-05 | Spark Connect GA, ANSI mode default, VARIANT type; SparkR deprecated[^7]. |

## References

[^1]: Apache Software Foundation, "The Apache Software Foundation Announces Apache Spark as a Top-Level Project" — 2014. https://www.apache.org/
[^2]: Zaharia et al., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing", NSDI 2012. https://www.usenix.org/conference/nsdi12/technical-sessions/presentation/zaharia
[^3]: Armbrust et al., "Spark SQL: Relational Data Processing in Spark", SIGMOD 2015; Catalyst optimizer. https://spark.apache.org/sql/
[^4]: Spark 2.0 / Project Tungsten — whole-stage code generation. https://spark.apache.org/releases/spark-release-2-0-0.html
[^5]: Spark 3.0 release notes — Adaptive Query Execution. https://spark.apache.org/releases/spark-release-3-0-0.html
[^6]: Spark Connect overview. https://spark.apache.org/docs/latest/spark-connect-overview.html
[^7]: Spark 4.0 release notes — ANSI mode default, Spark Connect GA. https://spark.apache.org/releases/spark-release-4-0-0.html

## Tags

scala, jvm, big-data, distributed-computing, data-engineering, sql, stream-processing, machine-learning, etl, apache, dataframe
