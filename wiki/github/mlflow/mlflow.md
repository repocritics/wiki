# mlflow/mlflow

> Open-source platform for the ML/AI lifecycle — experiment tracking, model packaging, a registry, and (since the 2.x/3.x era) LLM tracing and evaluation.

[GitHub repo](https://github.com/mlflow/mlflow) ·
[Official website](https://mlflow.org) ·
[License: Apache-2.0](https://github.com/mlflow/mlflow/blob/master/LICENSE.txt)

## Overview

MLflow was announced by Databricks at the 2018 Spark+AI Summit as a framework-agnostic way to track experiments and package models[^1]. It joined the Linux Foundation (LF AI & Data) in 2020[^2], which is why the project is vendor-neutral on paper despite Databricks employing most of the core maintainers and shipping a managed hosted version. For years its identity was the four classic components — Tracking, Projects, Models, and (from late 2019) the Model Registry — aimed at data scientists who wanted reproducibility without adopting a heavyweight platform.

Since 2024 the project has pivoted hard toward GenAI: one-line autologging/tracing for 60+ agent and LLM frameworks, an evaluation harness with LLM judges, a prompt registry, and an AI Gateway. The repo description and README now lead with "AI engineering platform for agents, LLMs, and ML models," and the current default branch reflects that GenAI-first framing. The defining tension is that MLflow tries to be one tool for two fairly different audiences — classical ML practitioners logging `sklearn`/`xgboost` runs, and LLM app teams wanting OpenTelemetry-style traces — and the surface area (and the ~2,000 open issues) reflects that breadth.

## Getting Started

```bash
pip install mlflow
mlflow server --host 127.0.0.1 --port 5000   # starts tracking server + UI
```

```python
import mlflow

mlflow.set_tracking_uri("http://127.0.0.1:5000")
mlflow.set_experiment("demo")

with mlflow.start_run():
    mlflow.log_param("lr", 0.01)
    for step, loss in enumerate([0.9, 0.6, 0.4]):
        mlflow.log_metric("loss", loss, step=step)
    mlflow.log_artifact("model.pkl")   # any file: weights, plots, configs
```

```python
# One-line tracing for an LLM app (2.x+)
import mlflow
mlflow.openai.autolog()   # every OpenAI call is now captured as a trace
```

Runs, params, metrics, and traces appear in the UI at `http://127.0.0.1:5000`.

## Architecture / How It Works

MLflow is a Python package plus a server process, and the server is really two pluggable stores glued behind a REST API and a React UI:

1. **Backend store** — an SQLAlchemy-backed relational DB (SQLite by default, MySQL/Postgres in production) or a local file store. It holds run metadata: params, metrics, tags, experiment structure.
2. **Artifact store** — a blob location (local FS, S3, GCS, Azure Blob, HDFS) for the large objects: model files, plots, datasets. The tracking DB stores only the artifact URI, not the bytes.

The **Model** abstraction is the conceptual core: a model is a directory (`MLmodel` YAML + files) that can advertise multiple *flavors* — e.g. a scikit-learn model also carries the generic `python_function` (pyfunc) flavor. `pyfunc` is the lowest-common-denominator interface every downstream tool (batch scoring, `mlflow models serve`, Docker/SageMaker/Kubernetes deployment) targets, which is what lets one packaging format deploy almost anywhere.

**Autologging** works by monkey-patching integration libraries at import time, so `mlflow.sklearn.autolog()` (or the GenAI `mlflow.<provider>.autolog()`) hooks framework internals to record params, metrics, or traces without explicit logging calls. **Projects** (`MLproject` files) reproduce runs in conda/virtualenv/Docker environments, though this component gets less attention than it once did. The **Model Registry** adds versioning, stage transitions, and aliases on top of logged models — and, importantly, requires a database backend; it does not work against the plain file store.

## Production Notes

- **The default file store is a trap for teams.** It cannot back the Model Registry and does not handle concurrent writers well. Any multi-user deployment should run a Postgres/MySQL backend from day one; SQLite will hit `database is locked` errors under concurrency.
- **Artifact access is a common footgun.** By default clients read/write the artifact store *directly*, so every user and serving box needs cloud credentials to the bucket. Running the server with `--serve-artifacts` proxies blobs through the tracking server instead — cleaner security boundary, but the server becomes a throughput bottleneck.
- **Auth is bolted on, not foundational.** For most of its life MLflow shipped with no authentication; a basic username/password plugin arrived in the 2.x line and is still relatively minimal. Real deployments front the server with a reverse proxy / SSO or use a managed offering.
- **Serialization reproducibility.** Models are typically pickled/cloudpickled with a captured environment (`conda.yaml` / `requirements.txt`). Loading a model whose library versions differ from the logging environment is the classic production failure; pin and rebuild the env rather than trusting cross-version unpickling.
- **UI degrades at scale.** Experiments with tens of thousands of runs make the runs table sluggish; teams often prune or shard experiments.
- **Schema migrations across majors can bite.** The tracking DB has its own migration path (`mlflow db upgrade`); upgrading across 1.x → 2.x → 3.x on a large shared DB warrants a backup and a staging rehearsal.
- **Two products in one repo.** Classical-ML features are mature and stable; the GenAI tracing/eval surface moves fast and has churned APIs release to release. Check the changelog before pinning to a GenAI feature in production.

## When to Use / When Not

**Use when:**
- You want open-source, self-hostable experiment tracking and model packaging without committing to a SaaS.
- You need a single artifact/registry format that deploys across batch, real-time, Docker, and cloud endpoints.
- You already log runs and now want LLM tracing/eval in the same tool rather than adding a second observability stack.
- Your org values the vendor-neutral, Linux Foundation governance and Apache-2.0 license.

**Avoid when:**
- You only need LLM observability — a purpose-built tracing/eval tool is lighter than standing up the full MLflow server.
- You want a polished, managed collaboration UI out of the box — hosted proprietary trackers are more turnkey.
- You need first-class data/pipeline versioning tied to git — MLflow tracks runs, not data lineage in a git-native way.
- You want strong built-in multi-tenant auth/RBAC without a proxy or a managed vendor.

## Alternatives

- wandb/wandb — proprietary SaaS experiment tracking; use when you want a richer managed UI and team collaboration and don't need self-hosting.
- langfuse/langfuse — open-source LLM observability/eval; use when your scope is only GenAI tracing, not the classical ML lifecycle.
- iterative/dvc — git-native data and model versioning; use when reproducibility should live in your repo and DAG rather than a server.
- aimhubio/aim — lightweight open-source experiment tracker; use when you want fast local run comparison without a heavy server.
- kubeflow/kubeflow — Kubernetes-native ML pipelines; use when orchestration on k8s matters more than a tracking UI.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announced | 2018-06 | Unveiled by Databricks at Spark+AI Summit[^1]. |
| 1.0 | 2019-05 | First stable API; tracking/projects/models. |
| 1.4 | 2019-10 | Model Registry introduced. |
| — | 2020-06 | Project joins the Linux Foundation[^2]. |
| 2.0 | 2022-11 | Recipes, pipelines refresh, API cleanup. |
| 2.x | 2024 | GenAI tracing + autolog for LLM/agent frameworks, LLM evaluation. |
| 3.x | 2025 | GenAI-first repositioning: agents, prompt registry, AI Gateway[^3]. |

## References

[^1]: Databricks, "Introducing MLflow: an Open Source Machine Learning Platform" — 2018-06-05. https://www.databricks.com/blog/2018/06/05/introducing-mlflow-an-open-source-machine-learning-platform.html
[^2]: LF AI & Data Foundation announcement that MLflow joined the foundation (2020). https://lfaidata.foundation/blog/2020/06/25/mlflow-joins-linux-foundation/
[^3]: MLflow documentation — GenAI / agents, tracing, and AI Gateway. https://mlflow.org/docs/latest/genai

## Tags

python, mlops, llmops, experiment-tracking, model-registry, llm-observability, machine-learning, apache-2.0, databricks, ai-engineering
