# hyperopt/hyperopt

> Distributed hyperparameter optimization in Python, built around Tree-of-Parzen-Estimators search over conditional, mixed-type spaces.

[GitHub repo](https://github.com/hyperopt/hyperopt) ·
[Official website](http://hyperopt.github.io/hyperopt) ·
License: non-standard BSD-style (GitHub reports NOASSERTION)[^1]

## Overview

Hyperopt is a Python library for minimizing an objective function over a search space that may mix real-valued, discrete, and conditional dimensions[^2]. It came out of James Bergstra's work at the intersection of machine learning and Bayesian optimization; the canonical reference is the 2013 ICML paper "Making a Science of Model Search"[^3], and the Tree-of-Parzen-Estimators (TPE) algorithm it popularized traces to a 2011 NeurIPS paper[^4]. For most of the 2010s it was the default answer to "how do I tune model hyperparameters without a grid search," and it is still embedded in a large amount of production ML tooling, most visibly as a first-class tuning backend in MLflow and as an adapter in Ray Tune.

The defining tension today is momentum versus maintenance. Hyperopt introduced ideas — define-by-run-ish stochastic search spaces, TPE, pluggable parallel trial storage — that the rest of the ecosystem absorbed and then improved on. The last PyPI release, 0.2.7, shipped in November 2021[^5]; the repository still receives occasional commits but no tagged release in years. Newer projects (Optuna in particular) offer a friendlier API, better pruning, richer visualization, and more active development, and have become the default for greenfield work. Hyperopt persists because it is battle-tested, because its search-space DSL is genuinely expressive for conditional spaces, and because ripping it out of existing pipelines is rarely worth it.

The search space is expressed with `hp.*` stochastic expressions (`hp.uniform`, `hp.loguniform`, `hp.choice`, `hp.quniform`, etc.). Crucially, `hp.choice` can nest further stochastic expressions, so entire sub-configurations only exist when a branch is selected — this conditional structure is what TPE is designed to exploit and is hyperopt's most durable advantage over flat grid/random search.

## Getting Started

```bash
pip install hyperopt
# optional extras: pip install 'hyperopt[MongoTrials,SparkTrials,ATPE]'
```

```python
from hyperopt import fmin, tpe, hp, Trials, space_eval

# Search space with a conditional (nested) branch
space = hp.choice('model', [
    ('svm',  {'C': hp.loguniform('C', -5, 5)}),
    ('tree', {'max_depth': hp.quniform('d', 2, 20, 1)}),
])

def objective(args):
    kind, params = args
    # ... train a model, return a scalar loss to MINIMIZE ...
    return some_validation_error(kind, params)

trials = Trials()                      # records every evaluation
best = fmin(objective, space,
            algo=tpe.suggest,          # or hyperopt.rand.suggest, atpe.suggest
            max_evals=100,
            trials=trials)

print(best)                            # raw indices, e.g. {'model': 0, 'C': 1.7}
print(space_eval(space, best))         # reconstructed args: ('svm', {'C': ...})
```

`fmin` always minimizes. To maximize a metric (accuracy, F1), return its negative. Returning a richer dict with `{'loss': ..., 'status': STATUS_OK, ...}` lets you attach diagnostics and mark failed trials with `STATUS_FAIL`.

## Architecture / How It Works

Hyperopt separates three concerns: the **search space** (a graph of stochastic expressions), the **algorithm** (a `suggest` function), and the **trial store** (a `Trials` object). `fmin` is the driver that ties them together.

- **Search space as a graph.** `hp.*` expressions build an expression graph, not concrete values. The algorithm samples this graph; `space_eval` and `sample` deterministically materialize a point given the graph plus a set of drawn indices. Because `hp.choice` branches can nest other `hp.*` nodes, the effective dimensionality of a sample is data-dependent — this is the "awkward search space" the README advertises.

- **TPE.** Rather than modeling `p(loss | params)` like a Gaussian-process approach, TPE models `p(params | loss)` by splitting observed trials into "good" and "bad" groups and fitting density estimators to each, then proposing points that maximize expected improvement under the ratio. It handles conditional/discrete dimensions naturally and scales roughly linearly in trials, but it treats dimensions largely independently and does not model interactions the way a GP would. The README notes hyperopt was *designed* to also host GP- and regression-tree-based optimizers, but those were never implemented in the core library[^2].

- **Algorithms available:** random search (`rand.suggest`), TPE (`tpe.suggest`), and Adaptive TPE (`atpe.suggest`, an extra dependency)[^2].

- **Trials and parallelism.** `Trials` holds the full history in memory. Two distributed backends swap it out: `SparkTrials` fans trials across an Apache Spark cluster, and `MongoTrials` uses MongoDB as a work queue that external `hyperopt-mongo-worker` processes poll. Both are asynchronous — TPE proposes based on whatever trials have completed so far, so parallelism trades some sample-efficiency for wall-clock speed.

## Production Notes

**Maintenance status is the first thing to weigh.** No tagged release since 0.2.7 (2021)[^5]. Bugfixes land irregularly. If you need active upstream support, a modern SciPy/NumPy story, or new features, this matters.

**MongoDB parallelism is the classic footgun.** `MongoTrials` requires a running MongoDB plus separately launched `hyperopt-mongo-worker` processes, and the objective function and all its closures must pickle cleanly to reach those workers. Unpicklable objects, notebook-defined functions, and version skew between driver and workers are the usual failure modes. `SparkTrials` avoids the standalone Mongo dependency but inherits Spark's own serialization and cluster-config complexity.

**In-memory `Trials` is the default and does not persist.** A crashed `fmin` loses history unless you pickle the `Trials` object yourself between runs or use one of the distributed stores. There is no built-in resumable on-disk trial database comparable to Optuna's RDB storage.

**Reproducibility** depends on passing an explicit `rstate` (a `numpy.random.Generator`/`RandomState`) to `fmin`. Without it, runs are not deterministic.

**Scaling limits of TPE.** It is strong on low-to-moderate dimensional, conditional spaces. On high-dimensional continuous problems, on spaces with strong parameter interactions, or when you need principled early-stopping/pruning of bad trials mid-training, hyperopt has little to offer — it has no native pruner. Frameworks like Optuna and Ray Tune (ASHA/median pruning) win decisively there.

**You will often meet it through a wrapper.** hyperopt-sklearn (scikit-learn model selection), hyperas (Keras), MLflow's `hyperopt` integration, and Ray Tune's `HyperOptSearch` adapter are common entry points. Behavior and gotchas often originate in hyperopt even when the surface API is someone else's.

## When to Use / When Not

**Use when:**
- Your search space is genuinely conditional/hierarchical (choice of model → model-specific params) and you want an optimizer that exploits that structure.
- You already run MLflow or Ray Tune and want TPE as the backend with minimal new dependencies.
- You have existing hyperopt pipelines that work and are stable — no reason to churn.

**Avoid when:**
- You're starting fresh and want active maintenance, define-by-run ergonomics, pruning, and dashboards — reach for Optuna.
- You need per-trial early stopping / successive halving on expensive training runs — use Ray Tune's schedulers.
- Your problem is smooth, low-dimensional, and continuous with a tight sample budget — a Gaussian-process optimizer (scikit-optimize, Ax) is more sample-efficient.

## Alternatives

- optuna/optuna — the de facto modern successor: define-by-run API, built-in pruners, storage backends, dashboards; use it for essentially all new tuning work.
- ray-project/ray (Tune) — use when you need to distribute trials at scale and want schedulers like ASHA for early stopping of bad runs.
- scikit-optimize/scikit-optimize — use when the space is low-dimensional and continuous and Gaussian-process sample-efficiency matters more than throughput.
- facebook/Ax (with BoTorch) — use for principled Bayesian optimization, multi-objective, and constrained experimentation.
- keras-team/keras-tuner — use when you're tuning Keras/TensorFlow models specifically and want tight framework integration.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013 | TPE / search-space ideas formalized in the ICML "Science of Model Search" paper[^3]. |
| 0.1.2 | 2019-02 | Late-0.1 line; Python 3 support maturing. |
| 0.2 | 2019-10 | Major line: `SparkTrials`, API cleanups. |
| 0.2.5 | 2020-10 | Bugfixes; `atpe` and Spark refinements. |
| 0.2.7 | 2021-11 | Latest PyPI release as of 2026; repo commits continue without new tags[^5]. |

## References

[^1]: GitHub license API reports the repository license as "Other" / NOASSERTION — the LICENSE file is a non-standard BSD-derived text that GitHub's classifier does not recognize. Verify terms directly before redistribution. https://github.com/hyperopt/hyperopt/blob/master/LICENSE.txt
[^2]: Project README — "Distributed Asynchronous Hyperparameter Optimization in Python." https://github.com/hyperopt/hyperopt
[^3]: Bergstra, Yamins, Cox (2013), "Making a Science of Model Search," ICML 2013. http://proceedings.mlr.press/v28/bergstra13.pdf
[^4]: Bergstra et al. (2011), "Algorithms for Hyper-Parameter Optimization," NeurIPS 2011. https://papers.nips.cc/paper/4443-algorithms-for-hyper-parameter-optimization.pdf
[^5]: PyPI release history for hyperopt (latest tagged release 0.2.7). https://pypi.org/project/hyperopt/#history

## Tags

python, hyperparameter-optimization, machine-learning, bayesian-optimization, tpe, automl, distributed-computing, model-selection, optimization, spark, mongodb
