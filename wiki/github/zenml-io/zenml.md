# zenml-io/zenml

> A Python framework that decouples ML/AI pipelines from the infrastructure they run on — write steps once, swap the backend (local, Kubernetes, SageMaker, Vertex) without rewriting code.

[GitHub repo](https://github.com/zenml-io/zenml) ·
[Official website](https://zenml.io) ·
[License: Apache-2.0](https://github.com/zenml-io/zenml/blob/main/LICENSE)

## Overview

ZenML is an MLOps orchestration framework, first released in 2020 by the Munich-based company of the same name[^1]. Its core proposition is a separation between **pipelines** (Pythonic workflow logic — training a model, running an eval, driving an agent loop) and **stacks** (the infrastructure that logic runs on: an orchestrator, an artifact store, a container registry, an experiment tracker). You write `@step` and `@pipeline` functions once, then reconfigure the stack to move the same code from a laptop to Kubernetes or a cloud managed service without touching the pipeline body.

The defining tradeoff is that ZenML is an abstraction *over* your existing tools, not a replacement for them. It does not run compute, store artifacts, or track experiments itself — it wraps MLflow, Kubeflow, SageMaker, Vertex, S3/GCS, and dozens of other integrations behind a uniform interface, adding automatic containerization, artifact versioning, lineage, and metadata tracking on top. The payoff is portability and reproducibility; the cost is another layer to learn (the stack model) and abstractions that occasionally leak when a backend behaves differently than the uniform API implies.

Since roughly 2024 the project has repositioned from "MLOps for classical ML" toward "one AI platform from pipelines to agents," adding LLM/agent-oriented examples and integrations (LangGraph, LiteLLM, Langfuse) alongside the original scikit-learn/PyTorch/TensorFlow lineage[^2]. As of 2026 it carries roughly 5.5k GitHub stars, is actively maintained (daily commits), and pairs an open-source core with a commercial ZenML Pro managed offering.

## Getting Started

```bash
pip install "zenml[server]"   # plain `pip install zenml` gives a slimmer client only
zenml init                     # scaffold a repository
zenml login                    # start/connect to a server
```

```python
# run.py — a minimal pipeline of two steps
from zenml import pipeline, step

@step
def load_data() -> dict:
    return {"features": [[1, 2], [3, 4]], "labels": [0, 1]}

@step
def train(data: dict) -> str:
    # fit a model on data["features"]; the return value is versioned as an artifact
    return "trained-model-v1"

@pipeline
def training_pipeline():
    data = load_data()
    train(data)

if __name__ == "__main__":
    training_pipeline()
```

Running `python run.py` executes on the **active stack**. By default that is a local orchestrator + local artifact store; `zenml stack register`/`set` swaps in remote backends without editing the code above.

## Architecture / How It Works

ZenML uses a **client-server architecture**[^3]. The client is the Python SDK and CLI; the server (ZenML Server) hosts the metadata store, the REST API, and a separate web dashboard ([zenml-io/zenml-dashboard](https://github.com/zenml-io/zenml-dashboard)). `pip install "zenml[local]"` runs both in-process for development; production deploys the server separately (a Helm chart on Kubernetes is the documented path) and clients connect via `zenml login <url>`.

Key concepts:

- **Steps** — `@step`-decorated functions. Inputs and outputs are typed; every output is serialized and versioned as an **artifact** through a **materializer** (the pluggable layer that maps a Python object to bytes in the artifact store).
- **Pipelines** — `@pipeline` functions that wire steps into a DAG. ZenML traces the data-flow between step calls to build the graph.
- **Stacks** — a named bundle of infrastructure components: an *orchestrator* (local, Kubernetes, Kubeflow, Airflow, SageMaker, Vertex), an *artifact store* (local, S3, GCS, Azure), plus optional container registry, experiment tracker, and model deployer.
- **Containerization** — for remote orchestrators, ZenML builds a Docker image of your code and dependencies so the same environment runs everywhere. Powerful, but also where most first-run friction lives (see Production Notes).
- **Metadata & lineage** — runs, artifacts, models, and their relationships are recorded in the server DB (SQLite locally, MySQL/managed SQL in production), queryable via SDK, dashboard, or the separate ZenML MCP server for natural-language querying.

The abstraction is deliberately thin at the logic layer (your step is just Python) and thick at the infra layer (the stack). Most of ZenML's value and most of its sharp edges both live in that infra abstraction.

## Production Notes

**Containerization is the top source of first-run pain.** Remote orchestrators require ZenML to build and push a Docker image with your code and dependencies. Getting the requirements resolution right (which packages, base image, private indexes, large ML wheels) is the most common onboarding blocker; expect slow iteration until image layers cache warm.

**The server is a stateful dependency for team use.** Local single-user work needs no server, but shared metadata, collaboration, and the dashboard require a deployed ZenML Server backed by a real SQL database plus remote artifact/container stores — a non-trivial piece of infrastructure to run and upgrade, not a stateless sidecar.

**Migrations have historically been disruptive.** The 0.20.0 release (late 2022) was a large rewrite that introduced the current server/dashboard architecture and required an explicit metadata migration; the project remains in the 0.x series and has shipped further breaking changes across minor versions[^4]. Pin your version, read release notes before upgrading, and back up the server database first — schema migrations run on server start.

**Leaky abstractions across backends.** The uniform stack API hides real differences in orchestrator semantics (scheduling, retries, resource requests, secrets). Code that runs locally can behave differently on Kubernetes/SageMaker/Vertex; debugging a remote run still requires understanding the underlying backend, so the abstraction reduces but does not eliminate infra knowledge.

**Open-source vs Pro boundary.** The Apache-2.0 core is fully functional and self-hostable. Some collaboration, RBAC, and managed-control-plane features belong to the commercial ZenML Pro tier; evaluate where that line falls before committing, especially for governance needs.

## When to Use / When Not

**Use when:**
- You want the same ML/AI pipeline code to run portably across local, Kubernetes, and cloud managed backends without a rewrite.
- You need artifact versioning, lineage, and reproducibility layered over tools you already use (MLflow, W&B, SageMaker) rather than replacing them.
- You are standardizing a team's MLOps practice and want one framework spanning classical ML and LLM/agent workflows.

**Avoid when:**
- You have a single environment and no portability need — the stack abstraction is overhead you will not recoup.
- You only need experiment tracking or a model registry; a focused tool (MLflow) is lighter.
- Your problem is general data orchestration, not the ML lifecycle — a data-first orchestrator fits better.
- You cannot tolerate 0.x breaking changes or the operational cost of running the server.

## Alternatives

- kubeflow/kubeflow — heavier, Kubernetes-native ML platform; use when you are all-in on K8s and want the full suite rather than a portable abstraction over it.
- mlflow/mlflow — experiment tracking, model registry, and packaging; use when you want tracking/registry without pipeline orchestration or a stack abstraction (ZenML integrates it).
- Netflix/metaflow — comparable Pythonic pipeline decorators; use when you are AWS-centric and want a more prescriptive, less pluggable path.
- dagster-io/dagster — general-purpose data orchestrator; use when your core problem is data pipelines and scheduling rather than the ML/AI lifecycle.
- iterative/dvc — Git-centric data and model versioning; use when versioning is the primary need and you do not want to run a server.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2020-11 | Open-sourced by ZenML GmbH; Pythonic pipeline abstraction[^1]. |
| 0.20.0 | 2022 | Major rewrite: ZenML Server + dashboard, new metadata store, breaking migration[^4]. |
| — | 2023–2024 | ZenML Pro managed offering; expanded cloud stack integrations. |
| — | 2024–2026 | Repositioning to "pipelines to agents": LLM/agent examples, LangGraph/LiteLLM integrations[^2]. |

*(ZenML remains in the 0.x series; consult the changelog for exact per-release dates.)*

## References

[^1]: ZenML documentation and repository history. https://docs.zenml.io/ · https://github.com/zenml-io/zenml
[^2]: ZenML README, "One AI Platform From Pipelines to Agents," and examples directory. https://github.com/zenml-io/zenml/tree/main/examples
[^3]: ZenML docs, "System Architectures." https://docs.zenml.io/getting-started/system-architectures
[^4]: ZenML changelog / release notes. https://docs.zenml.io/changelog

## Tags

python, mlops, llmops, machine-learning, pipeline-orchestration, ml-infrastructure, artifact-tracking, kubernetes, ai-agents, reproducibility, apache-2.0
