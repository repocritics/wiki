# dmlc/xgboost

> Regularized gradient-boosted decision trees with a C++ core and bindings for Python, R, JVM, and distributed backends — the default choice for tabular ML.

[GitHub repo](https://github.com/dmlc/xgboost) ·
[Official website](https://xgboost.readthedocs.io/) ·
[License: Apache-2.0](https://github.com/dmlc/xgboost/blob/master/LICENSE)

## Overview

XGBoost ("eXtreme Gradient Boosting") is a gradient-boosting library built around a C++ core with first-class bindings for Python, R, Java/Scala (JVM), and more. It came out of Tianqi Chen's work at the University of Washington and the DMLC (Distributed Machine Learning Community) group; the design is described in the 2016 SIGKDD paper "XGBoost: A Scalable Tree Boosting System"[^1]. It became widely known around 2015–2016 as the model behind a large share of winning Kaggle solutions on structured/tabular data, and that reputation still drives adoption.

Functionally it implements gradient-boosted decision trees (GBDT/GBM/GBRT): an additive ensemble where each new tree is fit to the gradient of a differentiable loss with respect to the current predictions. What distinguished XGBoost from earlier GBM implementations was the explicit L1/L2 regularization term in the objective, second-order (Newton) gradient steps, a sparsity-aware split-finding algorithm that handles missing values as a learned default direction, and a weighted quantile sketch for approximate split proposals on large data[^1]. These are engineering and formulation choices, not a new learning paradigm — the value is in a fast, well-regularized, production-hardened implementation.

The defining tension is that XGBoost is extremely capable but not turnkey: it exposes a large hyperparameter surface (tree depth, learning rate, subsampling, column sampling, minimum child weight, regularization terms, gamma) and rewards careful tuning. On modern hardware it competes closely with LightGBM and CatBoost, and the "which is best" answer is dataset-dependent rather than settled.

## Getting Started

```bash
pip install xgboost          # Python (bundles the compiled C++ core)
# R:   install.packages("xgboost")
# JVM: xgboost4j / xgboost4j-spark on Maven Central
```

```python
import xgboost as xgb
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split

X, y = load_breast_cancer(return_X_y=True)
X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2)

clf = xgb.XGBClassifier(
    n_estimators=500,
    learning_rate=0.05,
    max_depth=6,
    subsample=0.8,
    colsample_bytree=0.8,
    tree_method="hist",        # histogram-based; add device="cuda" for GPU
    early_stopping_rounds=50,
    eval_metric="logloss",
)
clf.fit(X_train, y_train, eval_set=[(X_val, y_val)])
print(clf.predict_proba(X_val)[:5])
```

The `XGBClassifier` / `XGBRegressor` classes are scikit-learn-compatible (they slot into `Pipeline` and `GridSearchCV`). The lower-level native API uses `xgb.DMatrix` and `xgb.train`, which exposes a few options the sklearn wrapper does not.

## Architecture / How It Works

The core is a C++ library; every language binding is a thin wrapper over it, which is why behavior is consistent across Python/R/JVM and why the compiled wheel is large.

- **DMatrix** — the internal columnar data structure. Building a `DMatrix` copies and pre-bins data, so it is a one-time cost you pay before training. `QuantileDMatrix` pre-quantizes for the histogram method and materially reduces memory versus a plain `DMatrix`.
- **Tree methods** — `hist` is the histogram-based algorithm: features are bucketed into a fixed number of bins so split-finding scans bins instead of raw sorted values. It is the default in recent versions and the recommended method[^2]. The older `exact` and `approx` methods still exist but are rarely the right choice at scale.
- **GPU** — modern versions decouple the algorithm from the device: set `tree_method="hist"` and `device="cuda"`. The historical `gpu_hist` tree method is deprecated in favor of the `device` parameter[^2].
- **Sparsity-aware splitting** — missing values (and structural zeros) are routed to a learned default branch at each node rather than imputed, so sparse and missing data are handled natively without preprocessing[^1].
- **Regularized objective** — the loss combines a second-order Taylor expansion of the user's differentiable loss with an L1/L2 penalty on leaf weights and a `gamma` complexity penalty per split. This is the formal difference from a "plain" GBM.
- **Distributed training** — a collective-communication layer (historically "Rabit") lets the same core run across Dask, Spark/PySpark, and Ray, partitioning data across workers. The distributed path is real but is a different operational surface than single-node.

## Production Notes

**Hyperparameter sensitivity is the main operator cost.** Defaults are reasonable but rarely optimal, and it is easy to overfit with deep trees plus a high learning rate. The standard recipe is a low `learning_rate` with `early_stopping_rounds` against a validation set to choose the effective number of trees, then tune `max_depth`, `min_child_weight`, `subsample`, `colsample_bytree`, and the regularization terms (`reg_alpha`, `reg_lambda`, `gamma`). Budget for a tuning loop (Optuna, or `GridSearchCV`/`RandomizedSearchCV`) rather than expecting good numbers out of the box.

**Memory.** A `DMatrix` holds a pre-binned copy of your data, so peak memory during `fit` is roughly your dataset plus the DMatrix, not just one of them. `QuantileDMatrix` and the `hist` method reduce this; for data that does not fit in RAM there is an external-memory path, which trades speed for footprint and has evolved significantly across recent major versions.

**Model serialization and version skew.** Since roughly the 1.0 line the recommended format is JSON/UBJSON via `model.save_model("model.json")`; the legacy binary format is deprecated. Do not rely on Python `pickle` for long-term or cross-version model storage — pickles are tied to library internals and break across upgrades, whereas `save_model`/`load_model` is the supported forward-compatible path.

**Categorical features** have native, non-one-hot support (`enable_categorical=True`), but it has spent time labeled experimental and its interaction with encodings, `DMatrix` construction, and some tree methods is worth validating on your data before trusting it in production.

**GPU caveats.** GPU training is fast but adds a CUDA/driver dependency to your deployment surface, and exact reproducibility (bit-for-bit determinism) between CPU and GPU — and across GPU generations — is not guaranteed. Pin your XGBoost and CUDA versions if reproducibility matters.

**Upgrade pains.** Major versions have moved defaults and renamed things: `hist` became the default tree method, `gpu_hist` gave way to `device="cuda"`, and the serialization format shifted. Read the release notes before jumping a major version, and re-save models in the new format after upgrading rather than loading old artifacts blindly.

## When to Use / When Not

**Use when:**
- Your problem is tabular/structured data (classification, regression, ranking) where tree ensembles routinely beat deep nets.
- You need a mature, portable model with consistent behavior across Python, R, and JVM, or you need to train distributed on Spark/Dask.
- You have missing values or sparse features and want them handled without heavy preprocessing.
- You want a well-understood baseline that is hard to beat on structured data.

**Avoid when:**
- Your data is images, audio, text, or other high-dimensional unstructured input — use deep learning instead.
- You need a simple, dependency-light model inside an existing scikit-learn pipeline and don't want to tune much — `HistGradientBoostingClassifier` may be enough.
- You have very many high-cardinality categorical features and little appetite for tuning — CatBoost's defaults may get you there faster.
- Interpretability or a linear/monotonic model is a hard requirement (though XGBoost does support monotonic and interaction constraints).

## Alternatives

- microsoft/LightGBM — leaf-wise tree growth, often faster and lower-memory on large datasets; use when training speed and RAM on big data are the binding constraint.
- catboost/catboost — strong native categorical handling and good defaults; use when you have many categorical features and want less manual tuning.
- scikit-learn/scikit-learn — `HistGradientBoostingClassifier`/`Regressor` needs no extra dependency; use for smaller problems already living in an sklearn pipeline.
- dmlc/treelite — compiles trained tree models (including XGBoost) for fast, dependency-light inference; use when you need to deploy the model, not train it.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014 | Repository created under DMLC; grows out of UW research[^1]. |
| — | 2016 | SIGKDD paper published; peak Kaggle-era visibility[^1]. |
| 1.0.0 | 2020-02 | API stabilization; JSON model format direction, legacy binary format deprecated. |
| 2.0.0 | 2023 | `hist` as default direction, `device` parameter, multi-target/vector-leaf work[^2]. |
| 3.0.0 | 2025 | External-memory and large-scale training improvements (see release notes). |

## References

[^1]: Tianqi Chen and Carlos Guestrin, "XGBoost: A Scalable Tree Boosting System," 22nd SIGKDD, 2016. https://arxiv.org/abs/1603.02754
[^2]: XGBoost documentation — Tree Methods and GPU support. https://xgboost.readthedocs.io/en/stable/treemethod.html
[^3]: XGBoost release notes / changelog. https://xgboost.readthedocs.io/en/latest/changes/index.html

## Tags

machine-learning, gradient-boosting, gbdt, decision-trees, tabular-data, cpp, python, distributed-systems, gpu, kaggle
