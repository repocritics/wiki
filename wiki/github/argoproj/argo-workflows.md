# argoproj/argo-workflows

> Container-native workflow engine for Kubernetes — every step is a pod, every workflow is a CRD.

[GitHub repo](https://github.com/argoproj/argo-workflows) ·
[Official website](https://argo-workflows.readthedocs.io/) ·
[License: Apache-2.0](https://github.com/argoproj/argo-workflows/blob/main/LICENSE)

## Overview

Argo Workflows is a workflow engine that runs on Kubernetes and models each step of a job as a container. Workflows are defined as a Custom Resource Definition (CRD): you submit a `Workflow` object, a controller reconciles it, and each node in the graph becomes a pod. It originated at Applatix in 2017, was donated to the CNCF, and became a CNCF graduated project in December 2022[^1]. It is one of four projects under the Argo umbrella (alongside Argo CD, Argo Rollouts, and Argo Events).

The defining design choice — and its central tradeoff — is that Kubernetes *is* the runtime. There is no separate scheduler, worker pool, or executor cluster; the Kubernetes API server holds workflow state and pods are the unit of execution. This makes Argo a natural fit for teams already running Kubernetes for batch, ML, and CI/CD, and a poor fit for anyone who wants a workflow tool without operating a cluster. It also means workflow authorship is YAML-first: control flow (DAGs, loops, conditionals, retries) is expressed declaratively in the `Workflow` spec, not in a general-purpose programming language[^2].

Argo is most used for ML pipelines, data/batch processing, and infrastructure automation. It sits underneath higher-level tools — Kubeflow Pipelines and Netflix Metaflow both compile to Argo Workflows — and is commonly driven from Python via the Hera SDK rather than hand-written YAML[^3].

## Getting Started

Install the controller and (optional) API server/UI into a cluster:

```bash
kubectl create namespace argo
kubectl apply -n argo -f \
  https://github.com/argoproj/argo-workflows/releases/download/v3.6.0/quick-start-minimal.yaml
```

A minimal workflow — each `template` is a unit of work, `entrypoint` names the first one:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-
spec:
  entrypoint: main
  templates:
    - name: main
      container:
        image: busybox
        command: [echo]
        args: ["hello world"]
```

A DAG with dependencies (steps run in parallel except where `dependencies` force ordering):

```yaml
  templates:
    - name: pipeline
      dag:
        tasks:
          - name: extract
            template: run
          - name: transform
            template: run
            dependencies: [extract]
          - name: load
            template: run
            dependencies: [transform]
```

Submit with `argo submit --watch workflow.yaml` (CLI) or `kubectl create -f workflow.yaml`.

## Architecture / How It Works

The core is the **workflow-controller**, a Kubernetes controller that watches `Workflow` objects and drives them to completion. When a node becomes runnable it creates a pod; when the pod finishes the controller updates the workflow's `status.nodes` map and schedules the next nodes. State lives inside the `Workflow` object itself — there is no external state store by default.

Each step pod is composed of the user's **main** container plus a **wait** sidecar. Since v3.1 the default (and, in later 3.x, the only) executor is the **emissary executor**[^4]: instead of talking to the container runtime, `wait` re-executes the entrypoint via a shared `emissary` binary injected as an init container, then watches the process and captures outputs. This replaced the earlier docker/kubelet/k8sapi/pns executors, which each had runtime-specific coupling and permission requirements. A practical consequence: emissary needs to know the container's `command`, so images that rely on a Dockerfile `ENTRYPOINT` without an explicit `command` in the template can fail to start.

Template types include `container`, `script` (inline code), `resource` (apply/patch arbitrary Kubernetes objects), `dag`, `steps`, `suspend`, `http`, and `plugin`. Reusable logic is packaged as `WorkflowTemplate` / `ClusterWorkflowTemplate`, and scheduled runs use `CronWorkflow`. Artifacts (files passed between steps) are copied through an external artifact repository — S3, GCS, Azure Blob, Artifactory, HTTP, Git — by the wait container, not through the API server.

The optional **Argo Server** provides a REST/gRPC API and the React UI, handles SSO (OAuth2/OIDC), and serves the **workflow archive** (completed workflows persisted to Postgres/MySQL for later inspection). The controller and server are separate deployments; you can run the controller headless.

## Production Notes

**etcd pressure is the signature failure mode.** Because the full node graph lives in `status.nodes` inside the `Workflow` object, large or wide workflows push against etcd's ~1.5 MB per-object limit. The mitigations are all opt-in and worth configuring before you need them: **node status offloading** to a SQL database, **status compression**, and keeping fan-out widths bounded. A DAG with thousands of dynamically-generated tasks (`withItems` / `withParam`) is the classic way to hit this wall.

**Completed pods and workflows accumulate.** Without a **pod GC** strategy (`podGCStrategy`) and workflow **TTL** (`ttlStrategy`), finished pods and `Workflow` objects pile up and degrade both the cluster and the controller's list/watch performance. Configure both from day one.

**Controller scaling is vertical and sharded, not horizontal.** The controller runs active/passive via leader election — the standby does no work. Throughput is tuned with `--workflow-workers`, `--pod-workers`, and qps/burst flags against the API server; multi-tenant setups shard by `instanceID` rather than adding controller replicas. Very high pod-churn workloads can saturate the Kubernetes API server before Argo itself is the bottleneck.

**Synchronization is coarse.** Global concurrency uses semaphores (a ConfigMap) and mutexes; `parallelism` caps concurrent nodes per workflow or namespace. These are cooperative controls, not hard scheduler guarantees.

**Upgrade caution.** Executor removal (docker/pns/etc. → emissary) between minor 3.x releases silently changed pod behavior for some images. CRD schema changes ship with releases and should be applied before rolling the controller. Read the release notes for every minor bump — API and manifest layout have shifted across the 3.x line.

**Security.** The controller and executor need broad RBAC (pod create/patch, and for `resource` templates, whatever the applied objects require). Multi-tenant clusters should scope workflows to service accounts per namespace; the `resource` template type is effectively arbitrary cluster access if left unrestricted.

## When to Use / When Not

**Use when:**
- You already operate Kubernetes and want batch/DAG/ML pipelines running as native pods.
- Each step is naturally a container and you want per-step images, resources, and retries.
- You need artifact passing, parameterization, loops, and scheduled (cron) runs declaratively.
- You want a substrate other tools compile onto (Kubeflow, Metaflow, Hera).

**Avoid when:**
- You don't run Kubernetes and don't want to — the cluster is a hard dependency.
- You want workflows defined in a general-purpose language with local debugging (prefer code-native engines).
- Your graphs fan out to thousands of nodes and you can't tolerate etcd/status tuning.
- You need a managed, zero-ops scheduler or a lightweight single-node tool.

## Alternatives

- apache/airflow — mature Python-defined scheduler with a huge operator ecosystem; use it when you want code-defined DAGs and rich integrations without making Kubernetes mandatory.
- tektoncd/pipeline — Kubernetes-native CRD engine focused on CI/CD; use it when the workload is build/test/deploy rather than general batch or ML.
- kubeflow/pipelines — ML-specific layer with experiment tracking that historically compiles onto Argo; use it when you want the Kubeflow ecosystem, not a general engine.
- temporalio/temporal — durable, code-defined workflows with a dedicated backend; use it when you need long-running stateful orchestration written in a real language, not container-per-step.
- prefecthq/prefect — Python-native orchestration with hybrid execution; use it when developer experience and dynamic, code-driven flows matter more than pod-per-step isolation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | Initial release at Applatix; CRD-based container workflows[^1]. |
| 2.0 | 2018 | Post-donation rework; DAG and steps templating matured. |
| 3.0 | 2021-03 | GA milestone; new React UI, workflow archive, API server[^5]. |
| 3.1 | 2021-05 | Emissary executor introduced as default[^4]. |
| 3.4 | 2022 | Legacy executors removed; plugin templates; UI/API changes. |
| — | 2022-12 | Argo project reaches CNCF graduated status[^1]. |
| 3.5 | 2023 | Continued 3.x maintenance line. |
| 3.6 | 2024 | Latest stable minor line as of writing. |

## References

[^1]: CNCF, "Argo Project graduation announcement" — 2022-12-06. https://www.cncf.io/announcements/2022/12/06/argo-becomes-a-graduated-cncf-project/
[^2]: Argo Workflows documentation — core concepts. https://argo-workflows.readthedocs.io/en/latest/workflow-concepts/
[^3]: Hera — Python SDK for Argo Workflows. https://hera.readthedocs.io/
[^4]: Argo Workflows docs, "Workflow Executors" (emissary). https://argo-workflows.readthedocs.io/en/latest/workflow-executors/
[^5]: Argo Workflows releases. https://github.com/argoproj/argo-workflows/releases

## Tags

go, kubernetes, workflow-engine, dag, batch-processing, mlops, ci-cd, cncf, cloud-native, orchestration, crd, data-pipelines
