# aimhubio/aim

> Self-hosted, open-source experiment tracker for ML — a local RocksDB-backed metadata store plus a query UI, positioned as the OSS alternative to Weights & Biases.

[GitHub repo](https://github.com/aimhubio/aim) ·
[Official website](https://aimstack.io) ·
[License: Apache-2.0](https://github.com/aimhubio/aim/blob/main/LICENSE)

## Overview

Aim is an experiment tracking tool: you log metrics, hyperparameters, images, audio, distributions, and arbitrary metadata from training runs, and Aim gives you a web UI to group, compare, and query those runs, plus a Python SDK to read the same data back programmatically. It is developed by AimStack (aimhubio) and has been in development since 2019[^1]. The defining stance is self-hosted and open — all data lives in a local `.aim` repository directory on disk that you own, with no hosted-service dependency, which is the axis on which it competes with the SaaS-first incumbents (Weights & Biases, Comet, Neptune).

The design bet that separates Aim from TensorBoard is storage. Rather than reading TensorFlow event files, Aim maintains its own embedded database (built on RocksDB via the `aimrocks` binding) so that queries can span tens of thousands of runs without loading everything into memory[^2]. The stated target is handling "10,000s of training runs" in a single repo, and the UI's Explorers are built to slice that scale. The tradeoff is that this is a single-node, single-writer-oriented store: the happy path is one machine (or a shared filesystem) accumulating a repo, not a horizontally-scaled multi-tenant service.

Aim occupies a middle position — heavier and more capable than TensorBoard, lighter and less turnkey than a hosted platform or a full MLOps suite like ClearML. It tracks experiments well; it does not do model registry, pipeline orchestration, or artifact/dataset versioning, and it does not pretend to.

## Getting Started

```bash
pip install aim
```

```python
from aim import Run

run = Run(experiment="mnist-baseline")
run["hparams"] = {"lr": 1e-3, "batch_size": 64}

for step, (loss, acc) in enumerate(train_iter()):
    run.track(loss, name="loss", step=step, context={"subset": "train"})
    run.track(acc,  name="accuracy", step=step, context={"subset": "train"})
```

```bash
# from the directory containing the .aim repo
aim up          # launches the web UI (default http://127.0.0.1:43800)
```

The `context` dict is not decoration — it is how Aim distinguishes the same metric name across train/val/test so the UI can group and align them. Getting your context taxonomy right up front is what makes the Explorers useful later.

## Architecture / How It Works

The data model is small and worth internalizing:

- **Repo** — the `.aim` directory. All runs, metadata, and indexes live here. This is the unit of backup and the unit of "a tracking server serves one repo."
- **Run** — one execution. Holds params (a free-form dict tree) and **sequences** of tracked values.
- **Sequence** — a named, stepped series (a metric, or a stream of images/audio/text/distributions/figures) qualified by a `context` dict. `(name, context)` is the addressable key.

Storage sits on **RocksDB**, accessed through `aimrocks` (a Cython wrapper the project maintains). Aim layers its own structures on top: separate stores for run metadata, sequence values, and the index that the UI queries. This is why Aim can scan many runs quickly where a directory of event files cannot — but it is also why the storage format is Aim-specific and evolves across major versions.

Querying is done in **AimQL**, which is not a bespoke grammar but a restricted subset of Python evaluated against exposed objects (`run`, `metric`, `run.hparams.lr > 0.01 and metric.name == "loss"`). That makes filters expressive to Python users and awkward to anyone expecting SQL. The same query language backs both the UI search bar and the SDK's `Repo.query_metrics`-style access, so the mental model is shared.

Two deployment shapes exist. The default is **embedded/local**: the training process writes directly into the `.aim` repo on the same filesystem. The second is the **remote tracking server**, where `Run(repo="aim://host:port")` streams over gRPC to a central `aim server`. The remote path is the answer to "my training runs on ephemeral cluster nodes," but it is the less-travelled road and historically the source of more rough edges than the local path.

The frontend is a React application; the Explorers (Metrics, Images, Params, Scatters, etc.) are the product surface, backed by a FastAPI/Starlette server that translates AimQL into storage reads.

## Production Notes

- **Version 3 is a hard boundary.** Aim 3.0 (2021) was a ground-up rewrite of both storage and UI; repos and APIs from the 2.x line are not compatible, and most existing tutorials/answers online assume 3.x. Treat anything pre-3 as a different product[^3].
- **Single-writer storage.** RocksDB is embedded and not built for many processes concurrently writing the *same* repo on the *same* files. Highly parallel training (many workers each opening the repo) is where people hit lock contention and corruption reports; the intended pattern for that case is the remote tracking server, not N processes sharing one `.aim` over NFS.
- **Filesystem sensitivity.** Because it is RocksDB on disk, the repo does not tolerate networked/consumer filesystems well. NFS, SMB, and some overlay/container-volume setups have produced corruption and lock issues. Keep the repo on a local SSD; back it up by snapshotting the directory when nothing is writing.
- **Repo growth and cardinality.** High-frequency tracking (logging every step at high step counts, or many distinct `(name, context)` sequences) grows the store and slows the UI's cross-run queries. Down-sample what you log; the UI is comfortable with thousands of runs but degrades with pathological per-run sequence counts.
- **UI scale.** The Metrics Explorer rendering a very large number of series at once is a known soft spot — expect to lean on the query filter rather than loading everything and grouping in-browser.
- **No auth in the box.** `aim up` and `aim server` ship without authentication or multi-tenancy. Exposing them beyond localhost means putting a reverse proxy with auth in front; there is no built-in user/permission model.
- **Migration in, not out.** Aim provides importers (e.g. from TensorBoard/MLflow) to help adoption, but there is no first-class "export my whole repo to platform X." The `.aim` directory plus the SDK is your escape hatch.

## When to Use / When Not

**Use when:**
- You want experiment tracking that is fully self-hosted and open, with data you physically own.
- You run on one machine or a controlled cluster and log 100s–10,000s of runs you need to compare.
- Your team is Python-native and comfortable expressing filters as Python expressions.
- You've outgrown TensorBoard's cross-run comparison but don't want a SaaS bill or vendor lock-in.

**Avoid when:**
- You need managed, zero-ops tracking with built-in auth, teams, and SSO — a hosted platform saves you the operational surface.
- You need model registry, pipeline orchestration, or dataset/artifact versioning — that is MLOps-suite territory, not Aim's scope.
- Your writers are massively concurrent against shared storage and you can't run the remote server — the single-writer store will fight you.
- Your metadata must live on NFS/networked storage — the RocksDB backend is unhappy there.

## Alternatives

- mlflow/mlflow — pick when you want tracking plus model registry and deployment lifecycle in one tool, and can accept a heavier, less-focused UI.
- tensorflow/tensorboard — pick when you only need per-run scalar/image visualization and don't need to query across many runs.
- wandb/wandb — pick when you want a managed, collaboration-first hosted service and are fine with a proprietary backend and SaaS pricing.
- allegroai/clearml — pick when you want self-hosted but need orchestration, agents, and data management, not just experiment tracking.
- IDSIA/sacred — pick when you want minimal, config-centric run logging as a library and will bring your own dashboard.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2019 | Project started as an open-source experiment tracker[^1]. |
| 2.x | 2020–2021 | Pre-rewrite line; separate storage/UI from current Aim. |
| 3.0 | 2021 | Full rewrite: RocksDB-backed storage engine + new React Explorers UI[^3]. |
| 3.x | 2022–2026 | Ongoing: remote tracking server, more framework integrations, UI Explorer additions. |

## References

[^1]: Aim repository, created 2019-05-31. https://github.com/aimhubio/aim
[^2]: Aim documentation — storage and performance overview. https://aimstack.readthedocs.io/en/latest/
[^3]: Aim 3.0 rewrite — announcement and docs. https://aimstack.io/blog

## Tags

python, machine-learning, experiment-tracking, mlops, observability, rocksdb, self-hosted, data-visualization, ml-tooling, open-source
