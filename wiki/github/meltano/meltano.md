# meltano/meltano

> Declarative, code-first ELT that orchestrates the Singer ecosystem of open-source data connectors.

[GitHub repo](https://github.com/meltano/meltano) ·
[Official website](https://meltano.com/) ·
[License: MIT](https://github.com/meltano/meltano/blob/main/LICENSE)

## Overview

Meltano is a command-line data integration engine that extracts data from sources and loads it into destinations (the "EL" of ELT), configured entirely in a version-controlled `meltano.yml` file. Its distinguishing bet is the **Singer specification**: rather than shipping its own connectors, Meltano orchestrates community-maintained Singer *taps* (extractors) and *targets* (loaders) — standalone programs that speak a JSON-over-stdout protocol originally defined by Stitch Data[^1]. Meltano handles plugin installation, configuration, secrets, incremental-replication state, environments, and pipeline invocation on top of them.

The project began as an internal GitLab tool and was spun out into an independent company in mid-2021[^2] (the GitHub repository was created at that time). Its audience is data/analytics engineers who want their pipelines to live in Git — reviewed, diffed, and deployed like application code — rather than clicked together in a SaaS UI. Meltano itself performs no transformation; the "T" is delegated to dbt via a transformer plugin, and scheduling is delegated to Airflow, Dagster, or cron via utility plugins. Meltano is the orchestrating spine, not a monolith.

The defining tension is the Singer ecosystem itself. Meltano's connector count (advertised as 600+ via Meltano Hub[^3]) is inherited from a large, unevenly maintained community catalog. A first-party tap like `tap-github` may be excellent; a long-tail tap may be a years-old, single-maintainer repository with schema bugs. Meltano's quality is therefore bimodal — the engine is solid, but any given pipeline is only as good as the specific tap and target variant it pins.

## Getting Started

Meltano is a Python package (installed via `pipx`, `pip`, or `uv`) and also ships as a Docker image[^4]:

```bash
pipx install meltano          # or: uv tool install meltano
meltano init my-project
cd my-project
```

Add an extractor and a loader, then run the pipeline:

```bash
meltano add extractor tap-github
meltano add loader target-jsonl

meltano config tap-github set repositories '["meltano/meltano"]'
meltano config tap-github set auth_token $GITHUB_TOKEN

meltano run tap-github target-jsonl
```

Each plugin and its config is recorded declaratively in `meltano.yml`:

```yaml
plugins:
  extractors:
  - name: tap-github
    variant: meltanolabs
    pip_url: git+https://github.com/MeltanoLabs/tap-github.git
    config:
      repositories: ["meltano/meltano"]
  loaders:
  - name: target-jsonl
    variant: andyh1203
    pip_url: target-jsonl
```

## Architecture / How It Works

**The Singer contract.** A tap is any executable that emits Singer messages — `SCHEMA`, `RECORD`, `STATE` — as newline-delimited JSON on stdout. A target reads those messages on stdin and writes them to a destination. Meltano pipes one into the other (`tap | target`) and interposes on the `STATE` messages to persist replication bookmarks. Because the interface is process-level stdio, taps and targets can be written in any language, though most are Python.

**Plugin isolation.** Each plugin is installed into its own Python virtual environment under `.meltano/`. This avoids dependency conflicts between plugins (a tap needing an old `requests` and a target needing a new one coexist) but means installs are slow and disk-heavy, and the `.meltano/` directory is disposable build state, not source.

**`meltano.yml` and environments.** The project file declares plugins, their `pip_url`, `variant`, config, and `select`/`metadata` rules. Environments (`dev`, `staging`, `prod`) layer config overrides. Secrets are pulled from environment variables or `.env`, never committed. `meltano run` executes a sequence of plugins as "blocks"; `meltano el` (formerly `elt`) runs a single extract-load pair; `meltano invoke` runs a plugin directly for debugging.

**State backends.** Incremental replication depends on persisting the `STATE` bookmark between runs. By default state lives in Meltano's SQLite/PostgreSQL system database; it can also be backed by S3, GCS, Azure Blob, or the local filesystem via the state backend config. Losing or corrupting state silently reverts a stream to full-table replication.

**Stream selection and maps.** `select` rules (`meltano select tap-github '*.id'`) control which streams and properties flow through. Stream maps — configured on the tap or via the `meltano-map-transformer` — allow inline filtering, aliasing, and PII hashing without a separate transform step.

**The SDK.** The [Meltano SDK](https://github.com/meltano/sdk) (`singer-sdk`) is a separate framework for authoring taps and targets in Python; most modern MeltanoLabs connectors are built on it, which raises the baseline quality of first-party plugins above the historical Singer average.

## Production Notes

**Connector quality is the real operational risk.** The engine rarely breaks; individual taps do. Before committing to a source, check which *variant* you are pinning (Meltano Hub often lists several variants of the same tap — e.g. `meltanolabs` vs. legacy `transferwise` for `tap-postgres`), when its repo was last updated, and whether it is SDK-based. Pin `pip_url` to a specific commit or version for reproducibility; an unpinned `pip_url` can pull a breaking change on the next `meltano install`.

**Throughput ceiling.** The Singer protocol serializes every record to JSON and pipes it through stdout/stdin between two processes. This is robust and language-agnostic but is not a high-throughput CDC path — for very large tables or low-latency change capture, the serialization overhead dominates. Batch message support and the SDK's `BATCH` capability mitigate this for some tap/target pairs, but not universally. For heavy CDC workloads, log-based tools are a better fit.

**State and incremental footguns.** Replication method (`INCREMENTAL`, `FULL_TABLE`, `LOG_BASED`) is per-stream and must be supported by the specific tap. A tap that claims incremental but has a buggy bookmark can silently re-emit or skip rows. When schemas change upstream, some targets do not automatically migrate destination tables; you may need a full refresh (`meltano el --full-refresh` / clearing state).

**No built-in scheduler or UI (by design, now).** Earlier Meltano shipped a web UI and bundled Airflow; both were deprecated as the project committed to code-first CLI use. Modern deployments run `meltano run` under an external orchestrator (Airflow, Dagster, GitHub Actions, cron). Treat Meltano as a library invoked by your scheduler, not a standalone platform.

**Install and CI cost.** Because every plugin gets its own venv, cold `meltano install` in CI is slow. Cache `.meltano/` or bake plugins into a Docker image. The slim Docker image is recommended; the full image is large.

**Upgrade awareness.** Config keys, the `elt`→`el` command rename, and utility-plugin (Airflow/dbt) packaging have shifted across major versions. Read the migration guide when jumping majors, and always pin the Meltano version in `pyproject.toml` / the Docker tag.

## When to Use / When Not

**Use when:**
- You want pipelines as reviewable, diffable code in Git rather than a clicked-together SaaS UI.
- Your sources are already well covered by mature Singer taps (databases, common SaaS APIs).
- You want to self-host, avoid per-row vendor pricing, and keep data in your own infrastructure.
- You are already running dbt for transforms and an orchestrator for scheduling, and need the EL layer to slot in.

**Avoid when:**
- You need a point-and-click, ops-light experience with a managed UI and connector monitoring (a hosted platform fits better).
- Your critical sources only have stale, single-maintainer taps — you would inherit maintenance you did not sign up for.
- You need high-volume log-based CDC with minimal latency; Singer's stdio protocol is the wrong shape.
- You want extraction, transformation, and orchestration in one integrated product rather than an assembled toolchain.

## Alternatives

- airbytehq/airbyte — UI-first, larger managed connector catalog, heavier to self-host (Docker/K8s); use it when you want a platform and monitoring UI over code-first config.
- dlt-hub/dlt — Python library you embed directly in your own code, no Singer layer; use it when you want extraction as a library call inside an existing Python codebase.
- dbt-labs/dbt-core — the transformation layer Meltano itself delegates to; complementary, not a replacement for EL.
- slingdata-io/sling-cli — lightweight single-binary CLI for database/file EL; use it for straightforward warehouse-to-warehouse moves without the plugin ecosystem.
- singer-io/singer-python — the raw Singer spec and reference library; use it directly only if you are hand-rolling taps without Meltano's orchestration.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018 | Originates as an internal data tooling project at GitLab[^2]. |
| 1.0 | 2019–2020 | First stable line; Singer-based CLI, `meltano.yml` project model. |
| — | 2021-06 | Spun out of GitLab as an independent company; GitHub repo created[^2]. |
| 2.0 | 2022 | `meltano run` block-based pipelines, jobs, environments; web UI deprecation path. |
| 3.0 | 2023 | Major cleanup; `elt`→`el`, removal of legacy bundled components, code-first focus. |

Exact minor-release dates are omitted where not verified; consult the changelog for specifics[^5].

## References

[^1]: The Singer specification. https://github.com/singer-io/getting-started/blob/master/docs/SPEC.md
[^2]: Meltano, "Meltano spins out of GitLab" (independent company, 2021). https://meltano.com/blog/
[^3]: Meltano Hub — connector catalog. https://hub.meltano.com/
[^4]: Meltano installation guide. https://docs.meltano.com/getting-started/installation
[^5]: Meltano changelog. https://github.com/meltano/meltano/blob/main/CHANGELOG.md

## Tags

python, data-engineering, elt, data-integration, singer, data-pipelines, cli, connectors, dataops, extract-load, open-source
