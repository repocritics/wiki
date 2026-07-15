# dlt-hub/dlt

> A Python-embedded EL(T) library that infers schema from messy sources and loads it into warehouses and lakes — a code-first alternative to Singer/Airbyte-style connector platforms.

[GitHub repo](https://github.com/dlt-hub/dlt) ·
[Official website](https://dlthub.com/docs) ·
[License: Apache-2.0](https://github.com/dlt-hub/dlt/blob/devel/LICENSE.txt)

## Overview

dlt ("data load tool") is a Python library, not a server or a platform: you `pip install dlt`, write a normal Python script that yields data, and dlt handles schema inference, normalization of nested JSON into relational tables, and bulk loading into a destination[^1]. It occupies the "EL" slot of ELT — extract and load — and deliberately leaves transformation to downstream tools like dbt or SQLMesh, though it ships thin helpers for both. The project is maintained by dltHub, a VC-backed company, and released under Apache-2.0.

The defining bet is that data-loading should be library code you embed anywhere Python runs — a Colab notebook, an AWS Lambda, an Airflow task, a laptop cron — rather than a standalone service you operate. That makes it dramatically lighter than Airbyte or Meltano to adopt, and it is the reason dlt shows up in agent/LLM-assisted pipeline workflows: the surface is small enough that a script generator can target it[^2]. The tradeoff is that "you write the connector" — outside the maintained REST API and SQL-database source templates, extraction logic is your Python code, your rate-limiting, and your maintenance burden. There is no managed connector catalog with SLAs.

The second defining trait is aggressive schema handling. dlt unpacks arbitrarily nested JSON into parent/child tables, coerces types, evolves the schema as new fields appear, and records all of this as versioned schema state in the destination. This is what makes messy-API ingestion feel effortless — and also where most production surprises originate.

## Getting Started

```sh
pip install dlt
# destination extras, e.g.: pip install "dlt[duckdb]"  /  "dlt[bigquery]"
```

```python
import dlt
from dlt.sources.helpers import requests

pipeline = dlt.pipeline(
    pipeline_name="chess_pipeline",
    destination="duckdb",
    dataset_name="player_data",
)

@dlt.resource(write_disposition="replace")
def players(names=("magnuscarlsen", "rpragchess")):
    for name in names:
        resp = requests.get(f"https://api.chess.com/pub/player/{name}")
        resp.raise_for_status()
        yield resp.json()

load_info = pipeline.run(players())
print(load_info)  # inspect the load package, row counts, schema updates
```

The `@dlt.resource` decorator turns a generator into a loadable table; `@dlt.source` groups related resources. `write_disposition` is `append` (default), `replace`, or `merge` (upsert, requires a primary/merge key).

## Architecture / How It Works

A `pipeline.run()` is three decoupled stages, each of which persists intermediate artifacts to a local working directory so a failed run is resumable:

1. **Extract** — resources are iterated and their output is written to disk as "extract packages" (buffered files, not held in memory). This isolates source I/O from the rest.
2. **Normalize** — dlt infers a schema from the extracted data, flattens nested objects into related tables, coerces types, normalizes column names to snake_case, and writes "load packages" (typically JSONL or Parquet) shaped for the destination.
3. **Load** — load packages are bulk-loaded, ideally via the destination's native path (COPY, staged files, `INSERT` batches). Warehouses like BigQuery, Snowflake, Redshift, Databricks, and Athena use a staging bucket (S3/GCS/Azure) for the fast path.

State lives **in the destination**, not in a separate control DB. dlt maintains a `_dlt` schema with `_dlt_loads`, `_dlt_pipeline_state`, and `_dlt_version` tables; incremental cursors, schema hashes, and load history are stored there. This is elegant (no external metadata store) but has consequences: if you point a pipeline at a fresh dataset, its incremental memory is gone; if you truncate those tables, you can re-load duplicates.

Nested structures become child tables linked by `_dlt_parent_id`, `_dlt_id`, and `_dlt_list_idx` columns — dlt reconstructs a relational model out of JSON. Incremental loading (`dlt.sources.incremental`) tracks a cursor field (e.g. `updated_at`) in pipeline state and passes it back on the next run. **Schema contracts** control what happens when the shape changes: `evolve` (default — add new columns/tables silently), `freeze` (raise), `discard_row`, or `discard_value`, settable per table/column/data-type.

The maintained REST API source is config-driven: you declare endpoints, pagination, and auth as a `RESTAPIConfig` dict rather than writing request loops. The SQL-database source can use a pyarrow/ConnectorX backend to pull rows column-wise, which is markedly faster than row-by-row for wide tables.

## Production Notes

**Nested-JSON table explosion.** The headline feature is also the headline footgun. A deeply nested API response produces a fan-out of child tables you may not want. Control it with `max_table_nesting` on the source/resource, or pre-flatten with `add_map`. Unbounded nesting on high-cardinality payloads produces schemas with dozens of `__` child tables and slow normalization.

**Column-name normalization collisions.** Fields are coerced to snake_case; `fooBar` and `foo_bar`, or names differing only by punctuation, can collide into one column. The default naming convention truncates long names and is destination-aware. If identifiers matter, pin an explicit naming convention rather than relying on defaults, because changing it later reshapes the schema.

**Incremental state is destination-coupled.** Because cursors live in `_dlt_pipeline_state`, resetting or moving a pipeline can silently re-ingest or drop data. Use `dlt pipeline <name> drop` / `--drop-all` deliberately, and be aware that a fresh `dataset_name` starts incremental tracking from scratch.

**`merge` requires care.** Upserts need a `primary_key` or `merge_key`; without one, `merge` degrades toward append semantics. dlt supports SCD2 dimensions, but staging-based merge on warehouses needs the staging bucket configured or the fast path silently falls back to slower strategies.

**Memory and throughput.** Extract buffers to files, but the normalize step is CPU-bound and single-pipeline throughput is often gated there, not by network. Tune `data_writer` buffer/file sizes and enable parallelism (`extract`/`normalize`/`load` worker settings in config) for large loads; use the Parquet/pyarrow path where the destination supports it.

**Version pinning.** dlt follows semver but the maintainers explicitly recommend pinning to patch-level (`dlt~=1.x.y`) because minor releases can include automatic schema migrations[^1]. Read release notes before minor bumps.

**Contributing new destinations is discouraged.** The maintainers state new destinations are unlikely to be merged due to maintenance cost, steering contributors toward improving the SQLAlchemy destination instead[^3]. If your warehouse isn't supported, expect to write and maintain a custom destination yourself.

## When to Use / When Not

**Use when:**
- You want code-first ingestion embedded in scripts, notebooks, Lambda, or Airflow with no separate service to run.
- Your sources are messy REST APIs or SQL databases and you want schema inference, nesting, and evolution handled for you.
- You're loading into DuckDB, BigQuery, Snowflake, Postgres, Redshift, Databricks, Athena, or a filesystem/lake and want native bulk-load paths.
- You value Python-native control over rate-limiting, pagination, and auth more than a point-and-click catalog.

**Avoid when:**
- You need a managed connector catalog with vendor-maintained SLAs (Fivetran/Airbyte Cloud fit better).
- Your workload is real-time streaming; dlt is batch/micro-batch, not a stream processor.
- You mainly need transformations — that's dbt/SQLMesh territory; dlt is EL, not T.
- You require massive distributed extraction (Spark-scale); dlt runs in a single Python process (parallelized, but not a cluster).

## Alternatives

- airbytehq/airbyte — server-based EL platform with a large connector catalog and UI; use it when you want managed, config-driven connectors instead of writing Python.
- meltano/meltano — Singer-tap/target orchestrator; use it when your org has standardized on the Singer ecosystem and CLI-driven pipelines.
- airbytehq/PyAirbyte — Python library that runs Airbyte connectors in-process; use it when you want Airbyte's catalog without the server, in code.
- slingdata-io/sling-cli — lightweight Go CLI for database/file replication; use it for fast config-only DB-to-DB or file copies with no Python.
- dbt-labs/dbt-core — the transformation layer dlt intentionally omits; pair it downstream when you need in-warehouse modeling.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2022-01-26 | Initial dlt-hub/dlt repository. |
| 0.x | 2022–2024 | Long pre-1.0 series; API and schema engine matured. |
| 1.0 | 2024-10 | First stable major; semver commitment, migration to `MAJOR.MINOR.PATCH` policy.[^1] |
| 1.x | 2024–2026 | REST API source, SQL pyarrow/ConnectorX backends, expanded destinations, LLM-native workflow tooling. |

## References

[^1]: dlt README — installation, semantic-versioning and pinning policy, contribution notes. https://github.com/dlt-hub/dlt
[^2]: dltHub docs, "LLM-native workflow." https://dlthub.com/docs/dlt-ecosystem/llm-tooling/llm-native-workflow
[^3]: dlt CONTRIBUTING guidance — new destinations discouraged, SQLAlchemy destination preferred. https://github.com/dlt-hub/dlt/blob/devel/CONTRIBUTING.md

## Tags

python, data-engineering, elt, etl, data-loading, data-pipeline, schema-inference, data-warehouse, data-lake, incremental-loading, rest-api, ingestion
