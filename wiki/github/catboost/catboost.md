# catboost/catboost

> Gradient boosting on decision trees with native categorical-feature handling and ordered boosting to fight target leakage.

[GitHub repo](https://github.com/catboost/catboost) ·
[Official website](https://catboost.ai) ·
[License: Apache-2.0](https://github.com/catboost/catboost/blob/master/LICENSE)

## Overview

CatBoost is a gradient-boosted decision tree (GBDT) library open-sourced by Yandex in July 2017[^1]. It sits in the same category as XGBoost and LightGBM — the three libraries that dominate tabular machine learning — but differentiates on two design choices: native handling of categorical features (no manual one-hot or label encoding) and *ordered boosting*, a training scheme meant to remove the prediction-shift bias that ordinary GBDT introduces when it computes gradients on the same rows it trained on[^2].

The intended user is anyone doing supervised learning on tabular data — classification, regression, and ranking — where features are a mix of numbers and categories. Its reputation is "good accuracy with little tuning": the defaults are conservative and the categorical handling removes a whole preprocessing step that XGBoost and LightGBM historically required. It is a fixture in Kaggle competitions and in production ranking/scoring systems.

The defining tradeoff is training cost versus convenience. The machinery that makes CatBoost accurate out of the box — random permutations for ordered boosting, ordered target statistics for categoricals, and symmetric (oblivious) trees — makes CPU training slower than LightGBM on large numeric datasets. You trade raw training throughput for less feature engineering, less overfitting risk, and very fast inference. It remains a single-vendor project (Yandex) under Apache-2.0, actively maintained as of 2026.

## Getting Started

```bash
pip install catboost          # Python; also available via conda-forge
```

```python
from catboost import CatBoostClassifier, Pool

train = Pool(
    data=[[1, "cat"], [2, "dog"], [3, "cat"], [4, "dog"]],
    label=[0, 1, 0, 1],
    cat_features=[1],          # column 1 is categorical — pass it as-is, no encoding
)

model = CatBoostClassifier(
    iterations=300, depth=6, learning_rate=0.05,
    loss_function="Logloss", verbose=False,
)
model.fit(train)
print(model.predict([[5, "cat"]]))
```

The `cat_features` argument is the point: strings are handled internally via ordered target statistics. Passing a high-cardinality category as a plain number instead throws away the library's main advantage. Bindings also exist for R, C++, Java, the command line, and Apache Spark[^1].

## Architecture / How It Works

**Ordered boosting.** Standard GBDT estimates the gradient for each training example using a model that was itself trained on that same example. This leaks the target and biases the model — "prediction shift." CatBoost's ordered mode fixes a random permutation of the data and, for each example, scores it with a model trained only on the examples that precede it in that permutation. Several permutations are maintained to reduce variance. This is the theoretical contribution of the NeurIPS 2018 paper[^2] and the reason training does more work per iteration than a naive booster.

**Ordered target statistics for categoricals.** Instead of one-hot encoding, CatBoost replaces a category with a statistic derived from the target (roughly, the mean label for that category), but computed using only prior rows in the permutation to avoid leakage. It also generates feature *combinations* of categoricals greedily during tree construction, which captures interactions one-hot encoding would miss.

**Symmetric (oblivious) trees.** By default every node at a given depth uses the *same* split condition, so a tree of depth `d` is a full, balanced binary tree described by `d` (feature, threshold) pairs. This constraint acts as regularization and makes inference extremely fast — a prediction is essentially computing one index into a lookup table rather than walking an irregular tree. The cost is expressiveness: some problems are better served by asymmetric splits, and `grow_policy` can be switched to `Depthwise` or `Lossguide` at the expense of the inference-speed advantage.

**GPU training.** CatBoost has first-class CUDA support, including multi-GPU, and GPU training is often several times faster than CPU. Distributed training is available through Apache Spark and the CLI. Models export to standalone formats (C++, Python, ONNX, CoreML, PMML) for embedding in serving stacks[^1].

## Production Notes

**Training is slower than LightGBM.** On large, mostly-numeric datasets, LightGBM's histogram + leaf-wise growth usually trains faster. CatBoost's edge is accuracy-per-unit-tuning and categorical handling, not raw training speed. Budget accordingly, and prefer GPU training for anything large.

**GPU and CPU are not bit-identical.** Results differ slightly between the two backends, and a handful of features/loss functions are CPU-only. Pin your training backend for reproducibility, and don't assume a GPU-trained model reproduces CPU numbers exactly.

**Categorical handling has a cost.** The permutation-based target statistics add both compute and memory overhead versus simple encodings, and generated categorical *combinations* can blow up if you feed many high-cardinality columns. Declare only genuine categoricals; leave truly numeric columns numeric.

**Overfitting control is built in.** Use `eval_set` plus the overfitting detector (`early_stopping_rounds`) rather than guessing `iterations`. `Pool` objects are the efficient path for repeated training and cross-validation; recreating raw arrays each fit is wasteful.

**Inference is a strength.** Symmetric trees make prediction fast and the exported model formats are self-contained, so the serving story (mobile via CoreML, JVM services via the Java/Spark model, C++ embedding) is genuinely good. Feature importances support both fast built-in methods and SHAP values, the latter being considerably more expensive.

**Governance.** It is a Yandex project. Development is open and continuous, but roadmap and release decisions are effectively single-vendor — weigh that as you would any corporate-backed OSS dependency.

## When to Use / When Not

**Use when:**
- Your tabular data has meaningful categorical features and you want to skip encoding pipelines.
- You want strong accuracy without heavy hyperparameter tuning.
- Inference latency matters (fast symmetric-tree scoring, portable model export).
- You need CoreML/ONNX/C++ model export for embedding outside Python.

**Avoid when:**
- Training throughput on huge numeric datasets is the binding constraint — LightGBM is usually faster.
- Your data is purely numeric and low-cardinality, where CatBoost's differentiators add overhead without payoff.
- You need a large, vendor-neutral contributor community and governance (XGBoost's is broader).
- The problem isn't tabular — deep learning is the better tool for images, text, and audio.

## Alternatives

- dmlc/xgboost — the incumbent GBDT library; use it when you want the broadest ecosystem, tooling, and vendor-neutral governance, and don't lean on native categorical support.
- microsoft/LightGBM — leaf-wise histogram boosting; use it when training speed on large numeric datasets is the priority.
- scikit-learn/scikit-learn — its `HistGradientBoostingClassifier` is a solid, dependency-light GBDT; use it when you want no extra install and moderate scale.
- h2oai/h2o-3 — distributed JVM ML platform; use it when you need cluster-scale training and AutoML over a single library.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-07 | Open-sourced by Yandex; NIPS 2017 workshop paper[^3]. |
| — | 2018-12 | "Unbiased boosting with categorical features" published at NeurIPS 2018[^2]. |
| 1.0.0 | 2021-09 | First 1.x release; API stabilization. |
| 1.2.x | 2023–2024 | Continued 1.2 line; Python/GPU/build maintenance. |

## References

[^1]: catboost/catboost README and documentation. https://catboost.ai/docs/
[^2]: Prokhorenkova, Gusev, Vorobev, Dorogush, Gulin, "CatBoost: unbiased boosting with categorical features", NeurIPS 2018 / arXiv:1706.09516. https://arxiv.org/abs/1706.09516
[^3]: Dorogush, Ershov, Gulin, "CatBoost: gradient boosting with categorical features support", ML Systems Workshop at NIPS 2017. http://learningsys.org/nips17/assets/papers/paper_11.pdf

## Tags

machine-learning, gradient-boosting, gbdt, decision-trees, tabular-data, categorical-features, python, cpp, gpu, kaggle, classification, ranking
