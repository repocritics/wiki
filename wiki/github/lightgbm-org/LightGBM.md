# lightgbm-org/LightGBM

> A histogram-based gradient boosting framework that grows trees leaf-wise for speed and low memory — fast to train, easy to overfit.

[GitHub repo](https://github.com/lightgbm-org/LightGBM) ·
[Official website](https://lightgbm.readthedocs.io/en/latest/) ·
[License: MIT](https://github.com/lightgbm-org/LightGBM/blob/main/LICENSE)

## Overview

LightGBM (Light Gradient Boosting Machine) is a gradient-boosted decision tree (GBDT) library originating from Microsoft Research, introduced in the NeurIPS 2017 paper by Guolin Ke et al.[^1] Its core is C++ with an OpenMP-parallelized training engine; the day-to-day interface for most users is the Python package (scikit-learn-compatible `LGBMClassifier` / `LGBMRegressor` / `LGBMRanker`, plus a lower-level `lgb.train` API). Official R, C API, and command-line front ends ship from the same source tree. In March 2026 the project moved from the `microsoft/LightGBM` namespace to the independent `lightgbm-org/LightGBM` organization, still maintained by the original team.[^2]

Two design choices define LightGBM and separate it from its main rival, XGBoost. First, it bins continuous features into a fixed number of histogram buckets (default 255) and searches splits over buckets rather than raw values — this is what makes it fast and memory-light. Second, it grows trees **leaf-wise (best-first)**: at each step it splits the leaf with the largest loss reduction anywhere in the tree, rather than expanding level-by-level. Leaf-wise growth reaches lower training loss in fewer nodes, but produces deep, unbalanced trees that overfit small datasets unless explicitly constrained.

The result is a library that is often the fastest to train on large tabular data and a perennial winner in Kaggle-style competitions, at the cost of being more tuning-sensitive than its peers. It handles categorical features natively (no one-hot encoding required) and missing values without imputation, which are practical advantages that regularly go under-appreciated.

## Getting Started

```bash
pip install lightgbm
# conda-forge and CRAN packages also exist; GPU/CUDA builds require compiling from source
```

```python
import lightgbm as lgb
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split

X, y = load_breast_cancer(return_X_y=True)
X_train, X_valid, y_train, y_valid = train_test_split(X, y, test_size=0.2, random_state=42)

train_set = lgb.Dataset(X_train, label=y_train)
valid_set = lgb.Dataset(X_valid, label=y_valid, reference=train_set)

params = {
    "objective": "binary",
    "metric": "auc",
    "num_leaves": 31,        # keep < 2^max_depth to bound overfitting
    "learning_rate": 0.05,
}

model = lgb.train(
    params,
    train_set,
    num_boost_round=500,
    valid_sets=[valid_set],
    callbacks=[lgb.early_stopping(stopping_rounds=25)],  # callbacks, not kwargs, since v4.0
)
preds = model.predict(X_valid)  # returns probabilities; no automatic thresholding
```

## Architecture / How It Works

The training loop is standard gradient boosting — fit each new tree to the gradients of the loss — but three techniques make it distinct:

- **Histogram binning.** Continuous features are pre-bucketed into `max_bin` bins (default 255, so one byte per value). Split-finding scans histograms instead of sorted feature values, cutting both time and memory. Histogram subtraction (a parent's histogram minus one child's yields the sibling's) halves per-node cost.
- **GOSS (Gradient-based One-Side Sampling).** Keeps all large-gradient (under-fit) instances and randomly subsamples small-gradient ones, reweighting to stay unbiased. Reduces the rows scanned per split without the accuracy loss of uniform sampling. Note: GOSS is not the default boosting mode — plain GBDT is — and must be enabled with `boosting="goss"`.
- **EFB (Exclusive Feature Bundling).** Bundles mutually-exclusive sparse features (rarely nonzero together, as in one-hot output) into single features, shrinking effective feature count.

**Leaf-wise growth** is controlled by `num_leaves` (the primary complexity knob), `max_depth`, and `min_data_in_leaf`. Because a leaf-wise tree with `num_leaves` leaves can be far deeper than a level-wise tree of the same size, the standard guidance is to set `num_leaves` well below `2^max_depth`.

**Categorical features** are handled by the algorithm directly: for a categorical column, LightGBM sorts categories by their accumulated gradient statistics and finds an optimal partition (an approximation of Fisher's method), rather than requiring one-hot encoding. You mark columns via the pandas `category` dtype or the `categorical_feature` parameter.

The C++ core exposes a stable C API; the Python and R packages are thin wrappers over it (Python via `ctypes`, R via a compiled shared library). The `Dataset` object is a preprocessed, binned, immutable construct — once constructed, feature binning is fixed, which is why validation sets must be created with `reference=train_set` to share bin boundaries.

## Production Notes

**Overfitting is the default failure mode on small data.** With defaults, LightGBM will happily memorize a few-thousand-row dataset. The tuning levers that matter most: lower `num_leaves`, raise `min_data_in_leaf` (default 20 is often too low), add `feature_fraction`/`bagging_fraction`, and use `lambda_l1`/`lambda_l2`. On tiny datasets you may hit "No further splits with positive gain" warnings — that is `min_data_in_leaf`/`min_gain_to_split` refusing to grow, not a bug.

**Reproducibility is not free.** Multithreaded floating-point summation is non-associative, so runs can differ slightly across thread counts. For bit-reproducible results set `deterministic=true`, pin `force_row_wise=true` (or `force_col_wise`), fix seeds, and hold `num_threads` constant across runs. Expect a speed penalty.

**macOS install friction.** The compiled library depends on OpenMP; on macOS the system Clang ships without it, so `pip install lightgbm` historically failed or silently ran single-threaded until `libomp` was installed via Homebrew. Wheels have improved, but OpenMP remains the most common install-time footgun outside Linux.

**GPU is two different things.** The original OpenCL GPU backend and the newer CUDA backend are separate build targets, both requiring compilation from source (`cmake -DUSE_GPU=1` or `-DUSE_CUDA=1`) — the PyPI wheels are CPU-only. GPU speedups are real for large dense datasets but modest or negative for small/sparse ones, and not every feature is GPU-supported.

**Distributed training** (data-parallel and feature-parallel, plus a voting-parallel mode) exists and can scale near-linearly in favorable settings, but it is operationally heavier than single-machine use; most teams reach for Dask, Ray (`lightgbm_ray`), or Spark (`SynapseML`) integrations rather than wiring sockets by hand.

**Upgrade friction (v3 → v4).** Version 4.0 removed the long-deprecated `early_stopping_rounds` and `verbose_eval` keyword arguments from `lgb.train`/`cv` in favor of the `callbacks` API, and dropped Python 3.6. Code copied from older tutorials frequently breaks on this. The project uses EffVer versioning, so a major bump signals migration effort, not just new features.

## When to Use / When Not

**Use when:**
- You have medium-to-large tabular data and want the fastest training among mainstream GBDT libraries.
- Your features include high-cardinality categoricals you would rather not one-hot encode.
- You are competing on tabular accuracy and are willing to tune `num_leaves` and regularization.
- You need ranking (`LGBMRanker`, LambdaRank/NDCG objectives) out of the box.

**Avoid when:**
- Your dataset is small (a few thousand rows) and you will not invest in regularization tuning — CatBoost's stronger defaults or a level-wise learner are safer.
- You need turnkey reproducibility with no configuration.
- Your problem is not tabular (images, text, sequences) — deep learning dominates there.
- You want zero non-Python dependencies and are already inside scikit-learn — its `HistGradientBoostingClassifier` covers the common case.

## Alternatives

- dmlc/xgboost — the incumbent GBDT; level-wise by default (with an optional leaf-wise/histogram mode), broader platform maturity. Use instead when you want the most battle-tested option or its wider ecosystem integrations.
- catboost/catboost — ordered boosting and built-in target-based categorical encoding. Use instead when categoricals dominate and you want strong results with minimal tuning.
- scikit-learn/scikit-learn — `HistGradientBoosting*` is a LightGBM-inspired histogram learner. Use instead when you want a native sklearn estimator with no extra dependency.
- dmlc/treelite — not a trainer but a compiler for deploying LightGBM/XGBoost models as fast standalone predictors. Use alongside when inference latency matters.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2016-10 | First public release out of Microsoft Research. |
| NeurIPS paper | 2017-12 | "LightGBM: A Highly Efficient Gradient Boosting Decision Tree."[^1] |
| 3.0.0 | 2020-08 | Major release; API and build modernization. |
| 4.0.0 | 2023-07 | Removed deprecated `early_stopping_rounds`/`verbose_eval` kwargs; adopted callbacks; dropped Python 3.6. |
| Org move | 2026-03 | Repository moved from `microsoft/LightGBM` to `lightgbm-org/LightGBM`.[^2] |

## References

[^1]: Guolin Ke et al., "LightGBM: A Highly Efficient Gradient Boosting Decision Tree." NeurIPS 2017. https://proceedings.neurips.cc/paper/2017/hash/6449f44a102fde848669bdd9eb6b76fa-Abstract.html
[^2]: LightGBM README / namespace-migration notice. https://github.com/lightgbm-org/LightGBM/issues/7187
[^3]: LightGBM documentation (Features, Parameters, Parameters-Tuning). https://lightgbm.readthedocs.io/en/latest/

## Tags

machine-learning, gradient-boosting, gbdt, decision-trees, tabular-data, cpp, python, r, kaggle, distributed, gpu
