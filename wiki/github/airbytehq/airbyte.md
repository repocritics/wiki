# airbytehq/airbyte

> Open-source ELT platform with a 600+ connector catalog, built around a Docker-per-connector protocol.

[GitHub repo](https://github.com/airbytehq/airbyte) ·
[Official website](https://airbyte.com) ·
License: MIT (core) + Elastic License 2.0 (parts)[^1]

## Overview

Airbyte is a data-movement platform for ELT (Extract-Load-Transform) pipelines: it pulls records from a source (an API, database, or file store) and lands them in a destination (warehouse, lake, or database), leaving transformation to downstream tools[^2]. It was started in 2020 by Michel Tricot and John Lafleur, with the thesis that only open source can cover the "long tail" of data sources that a closed vendor like Fivetran will never prioritize. The headline number is the connector catalog — 600+ connectors — and the fact that you can fork or hand-write your own.

The defining tension is **breadth versus reliability**. The catalog is large because most connectors are community- or CDK-generated rather than deeply maintained; connector quality varies from production-grade (Postgres, Stripe, Salesforce) to fragile long-tail integrations that break on upstream API changes. Airbyte's answer is the Connector Development Kit — a no-code Connector Builder, a low-code YAML CDK, and a Python CDK — so that when a connector is missing or broken, you can build or patch it rather than wait. That is the whole bet: connectors as commodity artifacts, not hand-crafted assets.

The second tension is **operational weight**. Airbyte is not a library; it is a distributed system (multiple services, a Temporal orchestrator, a job database, an object store for logs) that runs each connector as an isolated Docker container. This buys strong isolation and language-agnostic connectors, but the platform is heavy to self-host relative to code-first alternatives. Airbyte's own answer to that weight is Airbyte Cloud (managed) and, increasingly, lighter entry points like PyAirbyte and the Agent SDK.

## Getting Started

Local install uses `abctl`, Airbyte's control CLI, which provisions a local Kubernetes (kind) cluster[^3]:

```bash
# macOS/Linux — install abctl and launch a local Airbyte
curl -LsfS https://get.airbyte.com | bash -
abctl local install
abctl local credentials   # prints the URL + generated password
# UI at http://localhost:8000
```

To run a single connector without the platform, use PyAirbyte, which executes the same source containers/packages from Python[^4]:

```bash
pip install airbyte
```

```python
import airbyte as ab

source = ab.get_source(
    "source-faker",
    config={"count": 1000},
    install_if_missing=True,
)
source.check()                       # validate config + connectivity
cache = source.read()                # read into a local DuckDB cache
df = cache["products"].to_pandas()
print(df.head())
```

## Architecture / How It Works

The core abstraction is the **Airbyte Protocol**[^5]: a source is any program that, when run, emits a stream of newline-delimited JSON `AirbyteMessage` objects on stdout (`RECORD`, `STATE`, `LOG`, `TRACE`, `SPEC`, `CATALOG`), and a destination is a program that consumes that stream on stdin. Because the contract is stdin/stdout JSON, connectors can be written in any language and are packaged as Docker images. This is the source of both the ecosystem breadth and the runtime overhead — every sync spins up containers.

Key moving parts on a full platform deploy:

- **Server / config API** — CRUD for sources, destinations, connections; backed by a Postgres config-and-jobs database.
- **Temporal** — the workflow engine that orchestrates sync attempts, retries, and scheduling. Airbyte does not implement its own durable scheduler; it delegates to Temporal, which is itself a service with its own storage.
- **Workers** — launch the source and destination containers (or Kubernetes pods), broker the record stream between them, and handle state persistence.
- **Object storage** (MinIO / S3 / GCS) — connector logs and state.
- **Webapp** — a React UI; the Connector Builder lives here.

**State and incremental sync.** Sources emit `STATE` messages (cursor values); Airbyte persists the latest checkpoint so the next run resumes incrementally. Database sources support CDC (change data capture) via Debezium embedded in the source connector, reading the WAL/binlog rather than polling.

**Transformation.** Airbyte is deliberately EL, not ELT-in-one-box. Early versions ran dbt-based "normalization" to unpack JSON blobs into typed columns; that was deprecated and replaced by in-destination **Typing and Deduping**, which the destination connector performs directly. Heavier modeling is expected to happen in dbt/SQL downstream.

## Production Notes

**It is resource-hungry.** A self-hosted Airbyte is several always-on services plus Temporal plus a database plus per-sync containers. Teams routinely underestimate the baseline footprint; running it on a single small VM will fall over under real connection counts. Production deployments use the Helm chart on Kubernetes, with an external Postgres and external object storage rather than the bundled ones[^3].

**Connector quality is bimodal.** The certified/"Airbyte-maintained" connectors are solid; the long tail is not uniformly so. Before committing a pipeline to a given source, check its support level and recent release cadence in the connector registry. A missing field, a changed upstream API, or a schema drift can silently break a community connector.

**Memory on large syncs.** Both source and destination containers can hit memory ceilings on wide rows or large batches. Tuning per-connector JVM/container memory limits and batch sizes is a recurring operational task; OOM-killed sync attempts are a common support topic.

**The deployment story has churned.** Airbyte moved from `docker-compose` to `abctl` (kind-based) as the recommended local path, deprecating the old compose flow. Older tutorials and blog posts still reference docker-compose and will mislead. Similarly, connector-defined normalization tutorials predate the Typing-and-Deduping switch. Verify any guide against the current docs before following it.

**Upgrades touch many surfaces.** A platform upgrade may require migrating the config database, updating the Helm values schema, and separately bumping connector versions (connectors version independently of the platform). Pinning connector versions and testing in a staging deploy before rolling forward avoids surprise breakages.

**Licensing is not pure OSS.** The `gh` API reports `NOASSERTION` because the repo mixes licenses: the core platform is MIT, but some components and the enterprise features are under the Elastic License 2.0 (ELv2), which restricts offering the software as a managed service to third parties[^1]. Read the license FAQ before building a commercial hosted offering on top of it.

## When to Use / When Not

**Use when:**
- You need many heterogeneous sources landed in a warehouse/lake and want a UI plus a maintained-ish catalog rather than hand-writing each pipeline.
- You want the option to build or fork a missing connector without vendor gatekeeping.
- You want self-hosting for data-residency/cost reasons and can operate Kubernetes.
- Your model is EL + downstream dbt, not one-tool transformation.

**Avoid when:**
- You want a lightweight, code-first pipeline in one process — PyAirbyte, dlt, or Meltano are far less to operate.
- You need only one or two well-supported sources — a managed vendor or a single script is cheaper than running the platform.
- You need sub-minute streaming/CDC latency as the primary requirement — Airbyte is batch/micro-batch oriented; a purpose-built streaming CDC tool fits better.
- You cannot allocate ops capacity for a multi-service Kubernetes deployment.

## Alternatives

- meltano/meltano — Singer-tap-based, code/CLI-first EL; use instead when you want a git-versioned, YAML-configured pipeline without a heavy platform.
- dlt-hub/dlt — Python library for EL that runs in-process; use instead when you want connectors as code inside your own app/DAG, not a separate system.
- fivetran (closed source) — fully managed connectors; use instead when you want zero-ops and will pay per-row and forgo self-hosting.
- singer-io/singer — the tap/target spec Airbyte's ecosystem partly overlaps; use instead when you only need the interchange standard, not an orchestrator.
- debezium/debezium — dedicated database CDC; use instead when change data capture is your only need and you want streaming into Kafka.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 (public) | 2020-09 | Initial open-source release; Docker-compose deploy, first connectors[^2]. |
| Low-code CDK | 2022 | YAML-based connector definitions to grow the catalog faster. |
| Typing & Deduping | 2023–2024 | In-destination normalization replaces dbt-based normalization. |
| PyAirbyte | 2024 | Python library to run connectors without the platform[^4]. |
| 1.0 | 2024-10 | General availability milestone; `abctl` as the recommended deploy path[^6]. |

## References

[^1]: Airbyte license FAQ (MIT core + Elastic License 2.0 for parts/enterprise). https://docs.airbyte.com/platform/developer-guides/licenses/license-faq
[^2]: Airbyte README and company blog on the open-source data-movement thesis. https://github.com/airbytehq/airbyte
[^3]: Airbyte deployment docs — `abctl` local install and Helm/Kubernetes for production. https://docs.airbyte.com/deploying-airbyte/
[^4]: PyAirbyte documentation. https://docs.airbyte.com/using-airbyte/pyairbyte/getting-started
[^5]: Airbyte Protocol specification. https://docs.airbyte.com/understanding-airbyte/airbyte-protocol
[^6]: Airbyte 1.0 announcement — 2024-10. https://airbyte.com/blog/airbyte-1-0
</content>
</invoke>

## Tags

python, elt, etl, data-integration, data-pipeline, connectors, cdc, self-hosted, data-engineering, airbyte-protocol
