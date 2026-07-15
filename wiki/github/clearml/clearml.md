# clearml/clearml

> An end-to-end MLOps suite — experiment tracking, data versioning, remote execution, pipelines, and serving — bolted onto Python training code with two lines.

[GitHub repo](https://github.com/clearml/clearml) ·
[Official website](https://clear.ml/docs) ·
[License: Apache-2.0](https://github.com/clearml/clearml/blob/master/LICENSE)

## Overview

ClearML is the open-source SDK half of a broader MLOps platform built by the company of the same name (formerly Allegro AI). It began life as **Trains** / Allegro Trains and was renamed ClearML in 2020[^1]. The pitch is breadth: instead of adopting one tool for tracking, another for data versioning, a third for orchestration, and a fourth for serving, ClearML aims to cover all of them under one API and one web UI. The `clearml` package in this repo is the client SDK; the server, agent, and serving components live in separate repos ([clearml/clearml-server](https://github.com/clearml/clearml-server), [clearml/clearml-agent](https://github.com/clearml/clearml-agent), [clearml/clearml-serving](https://github.com/clearml/clearml-serving)).

The defining feature — and the defining tension — is auto-capture. A single `Task.init()` call monkey-patches your ML framework, argparse, stdout/stderr, TensorBoard, matplotlib, and model save hooks to record everything with no further instrumentation[^2]. When it works, onboarding is genuinely two lines. When it doesn't (an unsupported framework version, a patched logger conflicting with another tool, or capture firing where you didn't want it) the magic becomes something you have to reverse-engineer. The tradeoff is convenience versus a large implicit surface area you don't control.

The project is actively maintained — the SDK sees regular commits and the surrounding platform is a commercial product with a hosted free tier and self-hosting option. It is a real, widely-deployed alternative to the MLflow/W&B axis, with a stronger orchestration story than either.

## Getting Started

```bash
pip install clearml
clearml-init   # paste credentials from app.clear.ml or your self-hosted server
```

```python
from clearml import Task

# Auto-captures git state, packages, hyperparameters, console, metrics, models
task = Task.init(project_name="examples", task_name="hello world")

logger = task.get_logger()
for epoch in range(10):
    logger.report_scalar("loss", "train", value=1.0 / (epoch + 1), iteration=epoch)

# Any TensorBoard/matplotlib/model-save call from here is captured automatically
```

Without a configured server, the SDK stays offline unless `CLEARML_NO_DEFAULT_SERVER=0` is set to opt into the public demo server (experiments there are world-readable — never send sensitive runs).

## Architecture / How It Works

ClearML has three runtime pieces, only one of which is this repo:

1. **The SDK** (`clearml`) — the client. `Task.init()` installs *binding* modules that patch supported frameworks (PyTorch, TensorFlow/Keras, XGBoost, LightGBM, scikit-learn, FastAI, Hydra, and others) plus argparse and the standard logging streams. Captured data is batched and sent to the server over REST.
2. **The Server** (`clearml-server`) — the backend the UI and SDK talk to. It is not a single service: it fronts an `apiserver`, a `fileserver` for artifacts, the web app, and three datastores — **Elasticsearch** (metrics/logs/scalars), **MongoDB** (experiment/task metadata), and **Redis** (state/queues). This is the heaviest operational dependency in the stack.
3. **The Agent** (`clearml-agent`) — a daemon that pulls queued Tasks, recreates their exact environment (git clone at the recorded commit, pip/conda install of the recorded packages, or a Docker image), and runs them. Because the SDK records enough to reconstruct a run, "clone experiment → enqueue → agent executes on a GPU box" works without you repackaging anything.

Data management (`clearml-data` / the `Dataset` class) is a content-addressed layer over object storage (S3, GS, Azure, or a shared filesystem): datasets are versioned as diffs and hydrated on demand. **Pipelines** are built from existing Tasks — a controller Task enqueues step Tasks, so a pipeline is orchestration over the same execution primitive rather than a separate engine. **HPO** wraps Optuna and other optimizers over the clone-and-enqueue mechanism. **Serving** (`clearml-serving`) is a separate Triton-backed system, not part of this SDK.

The whole design is coherent around one idea: everything is a Task, and a Task is reproducible because the SDK over-captures its context. The coupling cost is that the server is mandatory for almost anything beyond local logging, and the server is a stateful, multi-datastore deployment.

## Production Notes

- **The server is the real operational burden.** Elasticsearch + MongoDB + Redis + apiserver + fileserver is a lot to run for a small team. The Docker-Compose deployment is fine on one box; scaling it (Elasticsearch heap tuning, index growth, backups across three stores) is where self-hosters spend their time. Many teams use the hosted tier specifically to avoid this.
- **Auto-capture is a leaky abstraction.** Framework patching is version-sensitive; a new PyTorch/TensorFlow release can outrun the binding and silently stop capturing (or throw). Pin known-good combinations and verify captured metrics after upgrades rather than assuming.
- **Storage credentials live client-side.** The SDK uploads artifacts and models directly to your object storage using credentials in `clearml.conf` / env vars — the server stores references, not (by default) the blobs. Misconfigured buckets or missing credentials fail at upload time, per-machine.
- **Elasticsearch retention.** Scalars, console logs, and debug samples accumulate in Elasticsearch indefinitely unless you prune. High-frequency `report_scalar` loops can bloat storage fast; downsample on the client.
- **Agent reproducibility depends on honest environments.** The agent reinstalls the recorded package set; if your original run relied on system libraries, editable installs, or uncommitted files outside git, the remote re-run can diverge. Docker-mode agents are the reliable path for anything nontrivial.
- **Backward compatibility is a stated promise** and generally holds for logs/data across versions[^3], but SDK ↔ server version skew can still surface API mismatches — keep them roughly in step.

## When to Use / When Not

**Use when:**
- You want tracking, data versioning, and remote GPU orchestration from one vendor and one UI, not four glued-together tools.
- You need reproducible remote execution ("run this experiment on the cluster") without manually containerizing each job.
- You're willing to run (or pay for) the server to get the orchestration payoff.

**Avoid when:**
- You only need experiment tracking and want the lightest possible dependency — MLflow or a hosted tracker is less to operate.
- You can't run a three-datastore server and won't use the hosted tier.
- You dislike implicit monkey-patching and prefer explicit, auditable logging calls.
- Your framework/version sits outside the supported binding matrix and auto-capture is the main reason you'd adopt it.

## Alternatives

- mlflow/mlflow — use instead when you want lighter-weight, vendor-neutral tracking and model registry and don't need built-in orchestration.
- wandb/wandb — use instead when you want the most polished tracking UI and collaboration and accept a proprietary/hosted-first product.
- iterative/dvc — use instead when your priority is git-native data and pipeline versioning rather than a live tracking server.
- aimhubio/aim — use instead when you want a fast, fully open-source local experiment tracker without a heavy server.
- Netflix/metaflow — use instead when orchestration and cloud-scale pipelines matter more than experiment-tracking UI.

## History

| Version | Date | Notes |
|---------|------|-------|
| Trains 0.x | 2019 | Initial open-source release as Allegro Trains[^1]. |
| ClearML rename | 2020-12 | Trains rebranded to ClearML; company renamed from Allegro AI[^1]. |
| 1.0 | 2021-06 | First stable major; API stabilization[^3]. |
| 1.x | 2021–2026 | Ongoing: pipelines, `Dataset`, clearml-serving, session, fractional-GPU added across the suite. |

Consult the [PyPI release history](https://pypi.org/project/clearml/#history) and repo tags for exact point-release dates.

## References

[^1]: ClearML (formerly Allegro Trains) rename announcement. https://clear.ml/blog/stay-in-control-with-clearml-nee-trains/
[^2]: ClearML SDK — `Task.init` automatic logging. https://clear.ml/docs/latest/docs/fundamentals/task
[^3]: ClearML documentation and repo README backward-compatibility statement. https://github.com/clearml/clearml

## Tags

python, mlops, llmops, experiment-tracking, machine-learning, deep-learning, data-versioning, orchestration, pipelines, model-serving, hyperparameter-optimization
