# skypilot-org/skypilot

> A cross-cloud broker for AI workloads: write a job once, and SkyPilot finds the cheapest available GPUs across Kubernetes, Slurm, and 20+ clouds and runs it there.

[GitHub repo](https://github.com/skypilot-org/skypilot) ·
[Official website](https://docs.skypilot.co/) ·
[License: Apache-2.0](https://github.com/skypilot-org/skypilot/blob/master/LICENSE)

## Overview

SkyPilot is a framework for running, managing, and scaling AI workloads on any compute you can get credentials for. It came out of UC Berkeley's Sky Computing Lab (the successor to RISELab / AMPLab) and its design is described in the 2023 NSDI paper on intercloud brokering[^1]. The project is now developed by a company of the same name, but remains Apache-2.0 and community-run on GitHub[^2].

The core value proposition is portability plus cost. You describe a task once — resources, data to sync, setup commands, run commands — in a YAML file or the Python API. SkyPilot's optimizer then picks the cheapest region/cloud that actually has the requested accelerators available, provisions them inside *your* cloud account (BYOC), syncs your code, installs dependencies, and streams logs. If a region is out of capacity or a spot instance is preempted, it fails over to the next option automatically. The pitch to AI teams is a Slurm-like interface; the pitch to infra teams is one control plane over heterogeneous compute.

The defining tension is that SkyPilot is a *broker*, not a runtime. It does not make your training code distributed, does not manage experiment state, and does not replace your cluster scheduler — it provisions and babysits infrastructure and hands you a machine (or pod) with your code on it. Its leverage comes entirely from breadth of cloud/Kubernetes/Slurm support and from the failover/autostop machinery around ephemeral GPU capacity. Where that machinery is not needed (a single stable on-prem cluster, one cloud with reserved capacity), much of SkyPilot's reason to exist falls away.

## Getting Started

```bash
# Install with the extras for the clouds you actually use
uv pip install "skypilot[kubernetes,aws,gcp]"
sky check   # verify which clouds/K8s are enabled from your local credentials
```

A minimal task, `my_task.yaml`:

```yaml
resources:
  accelerators: A100:8   # 8x NVIDIA A100; add `use_spot: true` for preemptible

num_nodes: 1

workdir: ~/torch_examples   # synced to ~/sky_workdir on the cluster

setup: |
  cd mnist && pip install -r requirements.txt

run: |
  cd mnist && python main.py --epochs 1
```

```bash
sky launch my_task.yaml -c my-cluster   # provision + run; picks cheapest available infra
sky down my-cluster                     # tear it down (or rely on autostop)
```

## Architecture / How It Works

SkyPilot sits between your task spec and a set of **cloud adaptors**. Each supported backend (AWS, GCP, Azure, OCI, Kubernetes, Slurm, RunPod, Lambda, Nebius, and more) implements provisioning, and the **optimizer** ranks feasible (cloud, region, instance-type) tuples by price using a bundled catalog, then attempts them in order with **auto-failover** on capacity or quota errors[^3].

Several distinct constructs share the `sky` CLI:

- **Clusters** (`sky launch`) — long-lived, interactive; you SSH in, sync code, attach an IDE. **Autostop/autodown** runs a small daemon on the cluster itself so idle machines are stopped or deleted even if your laptop is offline.
- **Managed jobs** (`sky jobs launch`) — fire-and-forget batch jobs, typically on spot instances, supervised by a always-on **jobs controller** VM/pod that detects preemption and re-provisions + re-runs the job elsewhere.
- **SkyServe** (`sky serve`) — model serving with a controller that runs replicas across regions/clouds behind a load balancer, with autoscaling.

Earlier versions leaned on the **Ray** autoscaler for cluster provisioning; SkyPilot has since moved much provisioning to its own implementation, though Ray still appears in parts of the stack. More recent releases introduced a **client–server architecture** (an API server) so a team can share one control plane and centrally track clusters and jobs, rather than each engineer holding state on their own machine[^4]. Cluster and job metadata otherwise lives locally under `~/.sky`.

The task abstraction is deliberately thin: your `setup`/`run` are just shell. This is why SkyPilot runs unmodified PyTorch, Ray, vLLM, or Slurm-launched code — it does not impose a programming model, only a provisioning and lifecycle model.

## Production Notes

**The controllers are real infrastructure.** Managed jobs and SkyServe each launch a controller instance that stays up to supervise your workloads. It is small but non-zero cost, and a common surprise on cloud bills; it must be torn down (`sky jobs controller` / `sky serve down`) when idle. Forgetting it is the classic SkyPilot footgun.

**Failover only spans what you enabled.** The optimizer can only move a job to clouds where `sky check` passed. If you configured one cloud, a capacity error is a hard failure, not a failover — and you still need real quota in each target cloud for failover to help.

**Spot recovery is provisioning-level, not application-level.** On preemption, the jobs controller re-provisions and re-runs your `run` command from the start. It does not checkpoint your training for you; resumability is your code's responsibility (write checkpoints to a bucket, make `run` idempotent). Re-provision + re-sync + re-`setup` can add minutes per recovery.

**State locality.** In the single-user model, cluster/job state lives on the launching machine under `~/.sky`. Lose or wipe that host and you can orphan running cloud resources (still billing). Teams should adopt the API server so state is centralized; audit with `sky status --refresh` if state and reality drift.

**Pricing catalog can lag.** The optimizer chooses "cheapest" from a bundled catalog that is not always in sync with live cloud pricing, discounts, or reservations. Treat its choice as a heuristic, not a billing guarantee.

**Kubernetes specifics.** SkyPilot on K8s maps clusters to pods and needs appropriate RBAC, a GPU device plugin, and images compatible with its SSH-based execution model. Gang scheduling for multi-node jobs interacts with your cluster's own scheduler; on contended clusters, partially schedulable jobs can wait.

## When to Use / When Not

**Use when:**
- You need GPUs across multiple clouds or K8s clusters and want one interface and automatic failover when capacity is scarce.
- You run bursty or spot-heavy training/batch jobs and want automatic preemption recovery and idle-cleanup.
- You want a portable, vendor-neutral task spec to avoid per-cloud launcher scripts and lock-in.

**Avoid when:**
- You have a single stable cluster (one cloud region or one on-prem box) with reserved capacity — the failover/broker machinery buys little.
- You need in-application distributed primitives (actors, parameter servers, data-parallel orchestration) — that is Ray/PyTorch's job, not SkyPilot's.
- You want a full ML platform with experiment tracking, pipelines, and model registry built in — SkyPilot is infra provisioning, not MLOps.

## Alternatives

- dstackai/dstack — closest peer: open-source, multi-cloud/K8s GPU orchestration with a container-first model. Use when you prefer a container-native workflow over SkyPilot's shell-based tasks.
- ray-project/ray — use instead when your problem is *in-app* distributed compute (RL, data-parallel training, serving) rather than cross-cloud provisioning; the two are often layered together.
- kubeflow/kubeflow — use when you are all-in on a single Kubernetes cluster and want pipelines, notebooks, and training operators as a platform.
- determined-ai/determined — use when you want integrated experiment tracking, hyperparameter search, and distributed training scheduling as one product.
- volcano-sh/volcano — use when you only need batch/gang scheduling *inside* one Kubernetes cluster, not cross-cloud brokering.

## History

| Version | Date | Notes |
|---------|------|-------|
| Paper | 2023-04 | "SkyPilot: An Intercloud Broker for Sky Computing," NSDI '23[^1]. |
| 0.x | 2022–2025 | Iterative releases: more clouds, managed spot jobs, SkyServe, autostop. |
| 0.7 | 2024 | Client–server API architecture for team deployments[^4]. |
| 0.12 | 2026-03 | Slurm support, job groups for RL, agent skill, pool autoscaling[^5]. |

## References

[^1]: Zongheng Yang et al., "SkyPilot: An Intercloud Broker for Sky Computing," NSDI 2023. https://www.usenix.org/conference/nsdi23/presentation/yang-zongheng
[^2]: skypilot-org/skypilot, `LICENSE` (Apache-2.0). https://github.com/skypilot-org/skypilot/blob/master/LICENSE
[^3]: SkyPilot docs, "Auto-failover / auto-provisioning." https://docs.skypilot.co/en/latest/examples/auto-failover.html
[^4]: SkyPilot docs, "API Server / team deployment." https://docs.skypilot.co/en/latest/reference/api-server/api-server.html
[^5]: SkyPilot v0.12.0 release notes. https://github.com/skypilot-org/skypilot/releases/tag/v0.12.0

## Tags

python, ml-infrastructure, gpu, multicloud, kubernetes, spot-instances, job-scheduler, distributed-training, llm-serving, mlops, cost-optimization
