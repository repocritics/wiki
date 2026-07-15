# optuna/optuna

> A define-by-run hyperparameter optimization framework for Python — search spaces are built imperatively at runtime, not declared up front.

[GitHub repo](https://github.com/optuna/optuna) ·
[Official website](https://optuna.org) ·
[License: MIT](https://github.com/optuna/optuna/blob/master/LICENSE)

## Overview

Optuna is a black-box optimization framework, most commonly used to tune machine-learning hyperparameters. It originated at Preferred Networks and was introduced in a 2019 KDD paper, "Optuna: A Next-Generation Hyperparameter Optimization Framework"[^1]. The 1.0 release followed in January 2020[^2]. As of 2026 it is one of the most widely adopted tuning libraries in the Python ecosystem, framework-agnostic by design and used well beyond ML (any function returning a scalar can be optimized).

Its defining choice is the **define-by-run** API. Instead of declaring a search space as a static dictionary (the "define-and-run" style of Hyperopt or scikit-optimize), you call `trial.suggest_*` methods inside the objective function itself. The search space is therefore constructed dynamically each trial, which makes conditional and nested spaces (choose an algorithm, then only sample the hyperparameters that algorithm needs) natural Python control flow rather than a declarative DSL. The tradeoff is that the search space is not knowable before running — some samplers and tooling need at least one trial to observe its shape, and static analysis of the space is impossible.

The library is deliberately thin at its core (an objective function, a sampler, a pruner, and a storage backend) and pushes framework-specific glue into separate packages: `optuna-integration` for library callbacks, `optuna-dashboard` for the web UI, and OptunaHub for community-published samplers. This keeps `pip install optuna` small but means the interesting pieces are scattered across repos.

## Getting Started

```bash
pip install optuna
# or: conda install -c conda-forge optuna
```

```python
import optuna

def objective(trial):
    # Search space is built imperatively, per trial.
    x = trial.suggest_float("x", -10, 10)
    y = trial.suggest_int("y", 0, 5)
    return (x - 2) ** 2 + y          # value to minimize

study = optuna.create_study(direction="minimize")
study.optimize(objective, n_trials=100)

print(study.best_params)   # e.g. {"x": 2.0007, "y": 0}
print(study.best_value)
```

Persist trials to a database (required for resumption and distributed runs):

```python
study = optuna.create_study(
    storage="sqlite:///study.db",
    study_name="demo",
    load_if_exists=True,
)
```

## Architecture / How It Works

Four decoupled components make up a study:

1. **Sampler** — decides the next set of parameters. Default is **TPESampler** (Tree-structured Parzen Estimator), a density-ratio Bayesian method that scales to many parameters cheaply. Others: `CmaEsSampler` (CMA-ES, strong on continuous spaces), `GPSampler` (Gaussian-process Bayesian optimization), `NSGAIISampler`/`NSGAIIISampler` (evolutionary, for multi-objective), `QMCSampler`, `GridSampler`, `RandomSampler`, and `BoTorchSampler`. `AutoSampler` (via OptunaHub) picks a sampler heuristically from the problem shape.
2. **Pruner** — kills unpromising trials early by having the objective `trial.report(value, step)` intermediate results and calling `trial.should_prune()`. Default is `MedianPruner`; `HyperbandPruner` and `SuccessiveHalvingPruner` implement the ASHA/Hyperband family. Pruning only applies to objectives with a meaningful iteration axis (training epochs, boosting rounds).
3. **Storage** — where trials live. `InMemoryStorage` (default) is per-process. `RDBStorage` persists to any SQLAlchemy-supported database (SQLite, MySQL, PostgreSQL) and is the substrate for distributed runs. `JournalStorage` (file- or Redis-backed) is an append-only alternative that avoids RDB locking pathologies.
4. **Study / Trial** — the orchestration objects. `study.optimize()` is the common driver, but there is also an **ask-and-tell** interface (`study.ask()` / `study.tell()`) that inverts control so you own the loop — useful for optimizing things that do not fit inside a single Python callable (batch jobs, human-in-the-loop, external processes).

**Distributed optimization** is not a special mode: you run the same script on N workers, all pointed at one shared storage URL. Each worker pulls the current trial history from storage, asks its sampler for a suggestion, and writes results back. Concurrency is therefore mediated entirely by the storage layer, which is why storage choice dominates distributed behavior.

## Production Notes

**SQLite does not scale to real parallelism.** SQLite's database-level write lock means concurrent workers serialize and throw "database is locked" under contention. For anything beyond a handful of local processes, use PostgreSQL/MySQL via `RDBStorage`, or `JournalStorage` (which was designed partly to sidestep RDB lock behavior). SQLite is fine for single-process persistence and resumption.

**Thread-based `n_jobs` is deprecated.** `study.optimize(objective, n_jobs=-1)` runs trials in threads and is throttled by the GIL for CPU-bound objectives; it was deprecated and the recommended pattern is process-level parallelism (multiple processes/machines against shared storage). Do not expect linear speedup from `n_jobs` on CPU work.

**Parallelism costs sample efficiency.** Bayesian samplers like TPE are sequential by nature — each suggestion ideally sees all prior results. When many workers request suggestions simultaneously, they sample against a stale history. Optuna mitigates this with a "constant liar" strategy in parallel TPE, but more workers still means less-informed suggestions per trial; wall-clock speedup and sample efficiency trade against each other.

**Failed and stalled trials.** In distributed runs a crashed worker can leave a trial stuck in `RUNNING` forever. `RDBStorage` supports a heartbeat mechanism plus `RetryFailedTrialCallback` to detect and re-enqueue dead trials — this is opt-in and easy to forget.

**Reproducibility is partial.** Seeding a sampler (`TPESampler(seed=...)`) makes suggestions deterministic, but parallel trial ordering, pruning timing, and the objective's own nondeterminism (GPU nondeterminism, data shuffling) are not controlled by Optuna. Identical `best_params` across runs is not guaranteed under parallelism.

**API deprecations.** Older `suggest_uniform`, `suggest_loguniform`, and `suggest_discrete_uniform` were folded into `suggest_float(..., log=..., step=...)`. Code and tutorials predating Optuna 3.0 frequently use the removed forms.

**Dashboard and integrations are separate installs.** `optuna-dashboard`, `optuna-integration` (Keras/PyTorch Lightning/XGBoost/LightGBM callbacks), and OptunaHub each version independently from core; pin them together to avoid interface drift.

## When to Use / When Not

**Use when:**
- You need conditional or dynamically-shaped search spaces expressed as ordinary Python.
- You want a framework-agnostic tuner that plugs into any objective returning a scalar.
- You need early stopping of bad trials (pruning) tied to a training loop.
- You want to scale from a laptop to many workers by only swapping the storage backend.

**Avoid when:**
- You need a fully declarative, statically-analyzable search space (define-and-run tools fit better).
- Your bottleneck is orchestrating thousands of GPU trials across a cluster — Ray Tune's scheduler/placement story is more complete (and can drive Optuna as its search algorithm).
- You want tuning tightly integrated with experiment tracking and infra out of the box — W&B Sweeps or a managed platform may be less assembly.
- The objective is cheap and low-dimensional — a grid or random search without the framework overhead may suffice.

## Alternatives

- hyperopt/hyperopt — the original TPE library; define-and-run API, comparatively stagnant maintenance. Use when you already have a hyperopt search space and don't need dynamic spaces.
- ray-project/ray — Ray Tune provides cluster-scale scheduling and can use Optuna as its search backend. Use when distributed orchestration, not sampling, is the hard part.
- facebook/Ax — Adaptive Experimentation on top of BoTorch; research-grade Bayesian optimization. Use when you want principled GP-based BO and A/B-style experiment management.
- scikit-optimize/scikit-optimize — lightweight `skopt`, sklearn-style API, minimal maintenance. Use for small BO problems in an sklearn pipeline.
- keras-team/keras-tuner — Keras-native tuner. Use when you are entirely inside the Keras/TensorFlow stack.
- automl/SMAC3 — random-forest-based Bayesian optimization from the AutoML group. Use for algorithm configuration and AutoML research workloads.

## History

| Version | Date | Notes |
|---------|------|-------|
| KDD paper | 2019-08 | "A Next-Generation Hyperparameter Optimization Framework"[^1]. |
| 1.0 | 2020-01 | First stable release; define-by-run API, pruners, RDB storage[^2]. |
| 2.0 | 2020-07 | Multi-objective optimization, improved integrations. |
| 3.0 | 2022-08 | Multivariate/constant-liar TPE, constrained optimization, stabilized APIs. |
| 4.0 | 2024-09 | JournalStorage stabilized, OptunaHub, storage/performance work. |
| 4.9 | 2026-06 | Current release line[^3]. |

## References

[^1]: Akiba, Sano, Yanase, Ohta, Koyama, "Optuna: A Next-Generation Hyperparameter Optimization Framework," KDD 2019. https://doi.org/10.1145/3292500.3330701
[^2]: Optuna 1.0 release. https://github.com/optuna/optuna/releases/tag/v1.0.0
[^3]: Optuna 4.9.0 release notes. https://github.com/optuna/optuna/releases/tag/v4.9.0

## Tags

python, hyperparameter-optimization, machine-learning, automl, bayesian-optimization, define-by-run, distributed, black-box-optimization, tpe
