# mage-ai/mage-ai

> A notebook-style, block-based tool for building, running, and scheduling data pipelines — pitched as a friendlier alternative to Airflow.

[GitHub repo](https://github.com/mage-ai/mage-ai) ·
[Official website](https://www.mage.ai) ·
[License: Apache-2.0](https://github.com/mage-ai/mage-ai/blob/master/LICENSE)

## Overview

Mage is an open-source data pipeline (orchestration) framework first released in 2022 by Mage AI, a venture-backed startup[^1]. Its central idea is that a pipeline is a directed acyclic graph of "blocks" — each block is a single file (Python, SQL, or R) with a typed role: data loader, transformer, or data exporter. Blocks are authored in a browser-based, notebook-style editor that runs code interactively while writing the underlying files and YAML metadata to disk. The result is meant to be Git-committable code, not clicks in a hosted UI.

The recurring positioning is "Airflow, but you don't have to hate it": file-based DAGs, live block previews, built-in data integration (ELT via the Singer spec), dbt support, and one-command Docker setup. That framing is also the project's defining tension. Mage bundles authoring IDE, execution engine, scheduler, and connector catalog into one process, which is excellent for a laptop and a small team, and becomes a coordination problem at scale — the same tradeoff Airflow made, packaged more approachably.

The other tension is commercial. Since 2024 the vendor has steered energy toward **Mage Pro**, a hosted/enterprise product, and the OSS README now frames the open-source edition primarily as the "start local, scale when ready" on-ramp[^2]. The open-source repo remains Apache-2.0 and actively pushed to, but readers should weigh how much roadmap attention flows to the paid tier.

## Getting Started

```bash
# Docker (the supported path)
docker run -it -p 6789:6789 \
  -v $(pwd):/home/src \
  mageai/mageai:latest \
  mage start my_project
# open http://localhost:6789

# or pip
pip install mage-ai
mage start my_project
```

A transformer block is an ordinary decorated Python function; upstream block outputs arrive as positional args:

```python
# transformers/clean_orders.py
from pandas import DataFrame

@transformer
def transform(df: DataFrame, *args, **kwargs) -> DataFrame:
    df = df[df["amount"] > 0]
    df["day"] = df["ordered_at"].dt.date
    return df   # returned value is passed to downstream blocks

@test
def test_output(df, *args) -> None:
    assert df["amount"].min() > 0, "negative amounts leaked through"
```

## Architecture / How It Works

A Mage **project** is a directory tree: `pipelines/<name>/metadata.yaml` declares the block DAG, block code lives under `data_loaders/`, `transformers/`, `data_exporters/`, and connection secrets live in `io_config.yaml`. Because everything is files, pipelines version-control and diff cleanly — the single biggest structural advantage over UI-first tools.

Blocks pass data by return value. In-memory the handoff is a DataFrame (pandas, Polars, or Spark); on disk Mage persists block outputs as **partitioned Parquet variables** so a downstream block, a re-run, or the preview pane can read an upstream result without recomputing it. This variable store is what makes the notebook-style incremental editing feel live, and it is also a source of surprise: outputs are cached materializations, not just logical edges.

Pipeline **types** are distinct engines, not one abstraction:

- **Batch** — the default block DAG described above.
- **Data integration** — source→destination ELT built on the **Singer** tap/target spec, configured largely through generated YAML rather than block code.
- **Streaming** — long-lived consumers (Kafka, Kinesis, and similar) rather than scheduled runs.

At runtime a Mage instance is really several roles in one image: a Python web/API server that backs the React front end, a **scheduler** process that polls a metadata database for due triggers, and **executors** that run blocks. Executor backends include local subprocess, PySpark, Kubernetes, AWS ECS, and GCP Cloud Run, selected per block or pipeline. The metadata database is SQLite by default and PostgreSQL for anything real. The front end is a separate app served by the same instance; most operators never touch it directly.

## Production Notes

**SQLite is a trap past one machine.** The default metadata DB is fine for a laptop and will silently bottleneck (locking, no concurrent scheduler) the moment you run multiple workers or want durable history. Move to PostgreSQL before, not after, going to production.

**The scheduler is a poller.** Triggers fire from a process that queries the metadata DB on an interval; there is no exactly-once guarantee free of your own idempotency. Running two schedulers against one DB without care can double-fire. Plan for one scheduler and horizontal executors.

**The Parquet variable store grows.** Cached block outputs accumulate under the project's variables directory (local disk or object storage). High-frequency pipelines with large DataFrames will fill disk unless you configure retention / an object-store backend; this surprises teams who assumed blocks were stateless.

**Memory lives where the block runs.** With the local executor, a block that loads a large table pulls it into the server process's RAM. "It ran in preview" on a sampled DataFrame does not predict full-run memory. For genuinely large data, push work into Spark or the warehouse and keep DataFrames out of the Mage process.

**Upgrades and issue backlog.** The project pushes frequently and the open-issue count is high (600+), typical of an actively used but broad-surface tool spanning IDE, scheduler, and dozens of connectors. Pin the `mageai/mageai` image tag rather than tracking `latest`, and test connector-heavy pipelines after upgrades — data-integration source/target behavior is the most movement-prone area. Also weigh the OSS-vs-Pro split: some newer capabilities are demonstrated on the commercial tier.

## When to Use / When Not

**Use when:**
- A small-to-mid team wants file-based, Git-versioned pipelines without hand-writing Airflow DAGs.
- You value interactive, notebook-style authoring with live previews over pure code-first workflows.
- You need built-in ELT connectors and dbt execution in one tool with Docker setup.

**Avoid when:**
- You need battle-tested multi-tenant scale, a large operator ecosystem, and deep community/vendor support at enterprise size — Airflow or Dagster are safer.
- You want a strictly code-first, asset/typed-lineage model with software-defined tests as the core primitive (Dagster's model).
- Your need is purely EL data movement — a dedicated connector platform (Airbyte, Meltano) is more focused.
- Long-term reliance on OSS parity matters and you're uneasy about the OSS/Pro roadmap split.

## Alternatives

- apache/airflow — use instead when you need the industry-standard scheduler, the widest operator/provider ecosystem, and proven scale, and can accept Python-DAG boilerplate.
- dagster-io/dagster — use instead when you want a code-first, software-defined-asset model with typed lineage and first-class testing.
- PrefectHQ/prefect — use instead when you want lightweight Python-native flows with dynamic runtime DAGs and minimal ceremony.
- meltano/meltano — use instead when the job is Singer-based EL(T) data integration specifically, as a CLI/config-first tool.
- kestra-io/kestra — use instead when you prefer a declarative-YAML, language-agnostic orchestrator with a strong events/triggers model.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-05 | Repo created; early tool focused on data cleaning/transformation[^1]. |
| pipeline pivot | 2022 | Repositioned to block-based pipeline orchestration (loader/transformer/exporter). |
| — | 2022–2023 | Added data integration (Singer), streaming pipelines, dbt support, cloud executors. |
| Mage Pro | 2024 | Commercial hosted/enterprise tier introduced; OSS reframed as "start local"[^2]. |

*Mage does not publish conventional semantic version milestones in its README; the table reflects phase, not tagged releases.*

## References

[^1]: mage-ai/mage-ai repository, created 2022-05-16. https://github.com/mage-ai/mage-ai
[^2]: Mage OSS README, "Start local. Scale when you're ready." and Mage Pro positioning. https://github.com/mage-ai/mage-ai/blob/master/README.md
[^3]: Mage documentation. https://docs.mage.ai

## Tags

python, data-engineering, data-pipelines, orchestration, etl, elt, dbt, workflow-scheduler, singer, notebook-ui, airflow-alternative, self-hosted
