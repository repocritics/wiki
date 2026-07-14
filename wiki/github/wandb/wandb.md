# wandb/wandb

> The Python SDK for Weights & Biases — experiment tracking, artifact versioning, and hyperparameter sweeps that log to a hosted (or self-managed) backend.

[GitHub repo](https://github.com/wandb/wandb) ·
[Official website](https://wandb.ai) ·
[License: MIT](https://github.com/wandb/wandb/blob/main/LICENSE)

## Overview

`wandb` is the client library for Weights & Biases, an ML experiment-tracking
and model-management platform first released in 2017[^1]. The code in this repo
is the instrumentation layer: you add a few lines to a training script, and
metrics, hyperparameters, system stats, artifacts, and media are streamed to a
backend that renders them as dashboards, run comparisons, and sweep leaderboards.
It is one of the default tracking tools in the PyTorch/JAX/Keras research world
and integrates with most training frameworks (Hugging Face, Lightning, Keras,
XGBoost, and others).

The defining tension is that the SDK is MIT-licensed and open, but the product it
talks to is not this repo. The rendering, storage, collaboration, and registry
features live on `wandb.ai` (multi-tenant SaaS) or in the separately-distributed
W&B Server for Dedicated Cloud / self-managed deployments[^2]. So "wandb is open
source" is true for the client and false for the platform that gives the client
its value. Teams evaluating it should treat this repo as an SDK, and the hosted
service — with its pricing, storage quotas, and data-residency implications — as
the actual dependency. Marketing now positions the whole thing as "the AI
developer platform," and LLM-app tracing has moved to a sibling product, Weave.

## Getting Started

```shell
pip install wandb
wandb login   # paste an API key from wandb.ai/settings
```

```python
import wandb

config = {"epochs": 1337, "lr": 3e-4}

# `with` marks the run finished on exit, "failed" on exception.
with wandb.init(project="my-awesome-project", config=config) as run:
    for step in range(config["epochs"]):
        # ... training ...
        run.log({"accuracy": 0.9, "loss": 0.1})

    # Version a file (dataset or model checkpoint) as an Artifact.
    artifact = wandb.Artifact("model", type="model")
    artifact.add_file("model.pt")
    run.log_artifact(artifact)
```

Set `WANDB_MODE=offline` to run with no network and `wandb sync` the local run
directory later; `WANDB_MODE=disabled` turns logging into no-ops for tests.

## Architecture / How It Works

`wandb.init()` creates a **Run** and, crucially, does not do the network I/O in
your training process. It spawns a separate long-lived **service process** that
owns the socket to the backend; your process hands it records over IPC and keeps
training. This is what lets `run.log()` be effectively non-blocking even when the
network is slow. Since the `wandb-core` rewrite this service is a Go binary rather
than a Python subprocess, which reduced memory overhead and improved upload
throughput[^3].

Each run maintains a few distinct streams: **config** (write-once inputs),
**history** (the time series you `log()`), **summary** (the latest value per
key), and **system metrics** (GPU/CPU/memory sampled by a background thread).
Locally everything is buffered under a `wandb/` directory as an append-only record
that can be replayed with `wandb sync` — the offline story is built on this.

**Artifacts** are content-addressed, versioned blobs: files are hashed, deduped,
and referenced by a manifest, so re-logging an unchanged dataset does not
re-upload it. **Sweeps** are hyperparameter searches where a central controller
hands configurations to `wandb agent` workers that each launch a run. **Tables**
and media types (images, audio, video, point clouds, HTML, matplotlib/plotly
figures) are logged as structured objects the UI knows how to render.

The coupling worth understanding: the SDK is a thin, fast producer; all the value
(diffing runs, grouping, alerts, model registry, reports) is computed server-side.
You cannot self-host that server from this repo — W&B Server is a separate
distribution[^2].

## Production Notes

- **The service process is the usual footgun.** In notebooks, Ray/Slurm jobs, and
  forked workers it can be orphaned or hang on exit; a wrapping `with wandb.init()`
  or an explicit `run.finish()` matters more than it looks. Atexit finalization
  during an ungraceful crash sometimes leaves a run stuck "running."
- **Distributed training.** In DDP/multi-GPU, initializing a run on every rank is
  a common mistake — you get N duplicate runs. The intended patterns are init on
  rank 0 only, or use `group=`/`wandb.setup()` so ranks share a group. There is no
  single default that is right for every launcher.
- **Network coupling and buffering.** With the SaaS backend unreachable, logging
  buffers to disk and retries; long outages can grow the local `wandb/` directory
  and, in some configurations, stall shutdown. `WANDB_MODE=offline` is the safe
  choice for air-gapped or flaky-network training.
- **Cost lives in storage, not stars.** High-frequency `log()` of large media or
  frequent artifact versions drives both local disk use and backend storage
  quotas. Log images/tables at an interval, and prefer artifact references
  (pointers to S3/GCS) over uploading multi-GB datasets when you control the bucket.
- **Import weight and side effects.** `import wandb` pulls in a non-trivial
  dependency surface; in latency-sensitive or minimal environments this is felt.
- **Version skew with the server.** The SaaS backend updates continuously; very old
  pinned clients can drift from newer server behavior. The `wandb-core` transition
  also changed defaults across the 0.17–0.18 line, so upgrades in that range are
  worth reading release notes for rather than bumping blindly[^3].

## When to Use / When Not

**Use when:**
- You want experiment tracking, run comparison, and sweeps with near-zero setup and
  broad framework integrations.
- Team collaboration, shareable reports, and a hosted model registry are worth a
  managed dependency.
- You need media/table logging (samples, confusion matrices, generated outputs)
  rendered without building a dashboard yourself.

**Avoid when:**
- You require a fully open-source, self-hosted stack — the client is open, the
  platform is not; evaluate mlflow or aim instead.
- Data residency or air-gap rules forbid a third-party SaaS and you cannot run
  (and pay for) W&B Server.
- Your needs are a few scalar curves — TensorBoard or plain CSV/matplotlib is
  lighter and has no external service.

## Alternatives

- mlflow/mlflow — use instead when you want open-source, self-hostable tracking plus a model registry with no vendor account.
- aimhubio/aim — use instead when you want a fast, local-first tracking UI you fully control.
- comet-ml — use instead when you want a wandb-style managed platform from a different vendor.
- tensorflow/tensorboard — use instead when you only need lightweight in-process scalar/graph visualization.
- allegroai/clearml — use instead when you want experiment tracking bundled with orchestration and data management.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | First `wandb` PyPI release; experiment tracking client[^1]. |
| Artifacts | ~2020 | Content-addressed dataset/model versioning added. |
| Reports / Tables | ~2020–2021 | Server-rendered reports and structured table logging. |
| wandb-core | 2024 | Go-based service backend, opt-in via `wandb.require("core")`, then default across the 0.17–0.18 line[^3]. |
| Weave split | 2024 | LLM-app tracing/eval moved to the separate `weave` package[^4]. |

Versioning stays in the `0.x` range; per the README, the minor version is bumped
when a Python version is dropped, and minimum Python is supported at least six
months past its upstream EOL[^1].

## References

[^1]: wandb/wandb README and PyPI history. https://pypi.org/project/wandb/#history
[^2]: W&B hosting options (Multi-tenant Cloud, Dedicated Cloud, Self-Managed). https://docs.wandb.ai/platform/hosting
[^3]: `wandb-core` announcement / release notes on the Go backend. https://github.com/wandb/wandb/releases
[^4]: Weave — GenAI tracking, debugging, and evaluation. https://wandb.github.io/weave

## Tags

python, machine-learning, mlops, experiment-tracking, deep-learning, hyperparameter-optimization, model-versioning, observability, sdk, pytorch, tensorflow, reproducibility
