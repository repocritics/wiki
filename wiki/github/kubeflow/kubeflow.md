# kubeflow/kubeflow

> The umbrella project for running the machine-learning lifecycle on Kubernetes — now a metadata gateway to a family of independently-versioned subprojects.

[GitHub repo](https://github.com/kubeflow/kubeflow) ·
[Official website](https://www.kubeflow.org) ·
[License: Apache-2.0](https://github.com/kubeflow/kubeflow/blob/master/LICENSE)

## Overview

Kubeflow is a collection of Kubernetes-native tools for the machine-learning lifecycle: notebook environments, pipelines, distributed training, hyperparameter tuning, model registry, and (historically) serving. It was announced by Google at KubeCon Austin in December 2017, grown out of the way Google ran TensorFlow jobs internally, and its original pitch was "make it as easy to run ML on Kubernetes as it is inside Google"[^1]. It reached 1.0 in March 2020[^2] and was accepted as a CNCF incubating project in 2023[^3], moving governance from Google to a set of community Working Groups and a Steering Committee.

The single most important thing to understand about *this repository* is that it is no longer where most of Kubeflow lives. The README states it plainly: `kubeflow/kubeflow` now "serves primarily as a gateway to Kubeflow subprojects and shared project metadata," and real development happens in the individual subproject repos. The 15.8k stars and 2.7k forks accrued to this repo reflect its history as the original monorepo (it once held the Notebooks controller, Central Dashboard, Profile controller, and access-management webhook); the open-issue count of zero reflects the migration — issues now live in `kubeflow/pipelines`, `kubeflow/katib`, `kubeflow/trainer`, `kubeflow/manifests`, and others.

The defining tension of Kubeflow is scope versus cohesion. It is not one product; it is a portfolio of separately-versioned components glued together by a manifests repo. That makes it the most complete open-source ML platform you can self-host, and also the one where "install Kubeflow" is a genuinely hard multi-week platform-engineering task rather than a `helm install`.

## Getting Started

There is no meaningful "install `kubeflow/kubeflow`" — you install the Community Distribution from the manifests repo with kustomize. The canonical instructions ship a retry loop because apply ordering across CRDs is racy:

```bash
git clone https://github.com/kubeflow/manifests.git
cd manifests

# The documented install is a retry loop, not a bug — CRDs and the
# resources that use them are applied together and may need re-apply.
while ! kustomize build example | kubectl apply --server-side --force-conflicts -f -; do
  echo "Retrying to apply resources"; sleep 20;
done
```

A minimal Kubeflow Pipeline, authored with the Python SDK (`pip install kfp`), targets the Pipelines subproject rather than this repo:

```python
from kfp import dsl

@dsl.component(base_image="python:3.11")
def add(a: int, b: int) -> int:
    return a + b

@dsl.pipeline(name="add-pipeline")
def pipeline(a: int = 1, b: int = 2):
    add(a=a, b=b)

# Compile to Argo Workflow YAML, then submit to a running cluster.
from kfp import compiler
compiler.Compiler().compile(pipeline, "pipeline.yaml")
```

## Architecture / How It Works

Kubeflow is a set of Kubernetes controllers and CRDs, one cluster of them per concern:

- **Pipelines (KFP)** — the flagship. Defines DAGs of containerized steps; the Python DSL compiles to YAML that is executed by **Argo Workflows** under the hood[^4]. Ships a UI, an API server, and a metadata store (MLMD) for tracking runs and artifacts.
- **Katib** — hyperparameter tuning and neural-architecture search, expressed as `Experiment` CRDs that spawn trials.
- **Trainer / Training Operator** — distributed training via job CRDs (`PyTorchJob`, `TFJob`, `XGBoostJob`, MPI). Recent work has consolidated these under a v2 "Kubeflow Trainer" API.
- **Notebooks** — a controller that provisions per-user Jupyter/VS Code/RStudio pods, historically part of this repo.
- **Central Dashboard, Profiles, KFAM** — multi-tenancy: a `Profile` CRD provisions a namespace per user with RBAC and resource quotas; the dashboard is the single pane of glass. These are the components that most directly still trace back to `kubeflow/kubeflow`.
- **Model Registry** — a more recent addition for cataloguing model versions and stages.

Multi-tenancy and auth are layered on **Istio** — the default distribution routes all component traffic through the Istio ingress gateway and uses an auth service (Dex/OIDC) plus Istio `AuthorizationPolicy` for per-namespace isolation[^5]. This Istio dependency is deep: it is not optional in the reference manifests, and it is the source of a large share of install and upgrade breakage.

Serving is a notable *absence*. KFServing was renamed and spun out as **KServe**, an independent project with its own release cadence and (as of recent releases) its own governance — it is bundled by many distributions but is no longer a Kubeflow subproject in the strict sense[^6].

## Production Notes

**Installation is the hard part, and it never fully stops being hard.** The kustomize overlays assume specific versions of Kubernetes, Istio, cert-manager, and Dex. Version skew between the manifests and your cluster's Istio is the most common failure mode. Most serious operators do not run the raw `example` overlay in production — they adopt a packaged distribution (see below) or maintain a forked, pinned overlay.

**Upgrades are re-installs.** Because components are independently versioned and glued by manifests, there is no in-place `kubeflow upgrade`. Moving from one Kubeflow release to the next typically means standing up new manifests and migrating workloads and metadata, and MLMD/pipeline database migrations have historically been a sharp edge.

**Multi-tenancy is real but Istio-coupled.** The Profiles/namespace model genuinely isolates users, but every isolation guarantee runs through Istio sidecars and AuthorizationPolicies. Debugging a "notebook can't reach the pipeline API" problem almost always ends in Istio config, not Kubeflow config.

**Resource footprint.** A full distribution runs Istio, Dex, cert-manager, the pipeline API server plus MySQL/MinIO (artifact + metadata storage), the notebook and profile controllers, Katib, and the training operator. This is a heavy control plane; it is not appropriate for a single laptop cluster despite the minikube-friendly marketing of the early years.

**Use a distribution, not upstream, in production.** Vendors ship hardened, supported builds — AWS (Kubeflow on AWS), Google Cloud, Azure, Nutanix, and the community-maintained "Kubeflow Community Distribution" — that pin versions and add cloud IAM/storage integration. Running the bare upstream manifests long-term means signing up to be your own distribution maintainer.

## When to Use / When Not

**Use when:**
- You are already committed to Kubernetes and want ML workloads (training, pipelines, notebooks) to live on the same substrate as everything else.
- You need genuine multi-tenant, self-hosted, on-prem ML infrastructure with per-team isolation and no dependency on a managed SaaS.
- You want composability — adopt only Pipelines, or only Katib, without the whole stack.

**Avoid when:**
- You want experiment tracking or a model registry and nothing else — Kubeflow's operational cost is wildly disproportionate to that need.
- You don't run Kubernetes, or your team lacks a platform engineer who can own Istio and CRD lifecycle.
- You want a single vendor-supported product with one version number and an upgrade button — Kubeflow is a portfolio, not a product.

## Alternatives

- mlflow/mlflow — use instead when you want experiment tracking, a model registry, and packaging without a Kubernetes control plane.
- flyteorg/flyte — use instead when pipelines are the whole point and you want a single cohesive Kubernetes-native workflow engine rather than an assembled stack.
- ray-project/ray (with KubeRay) — use instead when distributed training/serving/tuning in one framework matters more than a broad platform.
- zenml-io/zenml — use instead when you want a portable MLOps framework that orchestrates *onto* backends (including Kubeflow) rather than being the backend.
- apache/airflow — use instead when your orchestration is general data/ETL work that happens to include ML, not ML-first.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announce | 2017-12 | Introduced by Google at KubeCon Austin[^1]. |
| 0.1 | 2018-05 | First tagged release; ksonnet-based deployment. |
| 1.0 | 2020-03 | First stable release; core components declared GA[^2]. |
| — | 2021 | KFServing renamed to KServe and spun out as an independent project[^6]. |
| — | 2023 | Accepted as a CNCF incubating project; governance moves to Working Groups[^3]. |
| — | 2024–2025 | Repo restructured into subprojects; `kubeflow/kubeflow` becomes a metadata gateway. Training consolidates under Kubeflow Trainer v2. |

## References

[^1]: Google Cloud blog, "Introducing Kubeflow" — 2017-12-21. https://cloud.google.com/blog/products/gcp/introducing-kubeflow-a-composable-portable-scalable-ml-stack-built-for-kubernetes
[^2]: Kubeflow blog, "Kubeflow 1.0: Cloud Native ML for Everyone" — 2020-03-02. https://blog.kubeflow.org/release/official/2020/03/02/kubeflow-1-0-cloud-native-ml-for-everyone.html
[^3]: CNCF, "Kubeflow becomes a CNCF incubating project" — 2023. https://www.cncf.io/projects/kubeflow/
[^4]: Kubeflow Pipelines documentation — backend built on Argo Workflows. https://www.kubeflow.org/docs/components/pipelines/
[^5]: Kubeflow docs, multi-user isolation via Istio and Profiles. https://www.kubeflow.org/docs/components/central-dash/profiles/
[^6]: KServe project (formerly KFServing). https://kserve.github.io/website/
[^7]: Kubeflow README, "Repository Role" — this repo as a gateway to subprojects. https://github.com/kubeflow/kubeflow

## Tags

machine-learning, mlops, kubernetes, ml-pipelines, distributed-training, kubernetes-operator, jupyter, model-serving, cncf, platform, hyperparameter-tuning
