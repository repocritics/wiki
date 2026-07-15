# scikit-learn-contrib/imbalanced-learn

> Resampling techniques for class-imbalanced datasets, built to plug into the scikit-learn API.

[GitHub repo](https://github.com/scikit-learn-contrib/imbalanced-learn) ·
[Official website](https://imbalanced-learn.org) ·
[License: MIT](https://github.com/scikit-learn-contrib/imbalanced-learn/blob/master/LICENSE)

## Overview

imbalanced-learn (imported as `imblearn`) is a Python package of over- and
under-sampling algorithms for datasets where one class heavily outnumbers the
others — fraud detection, rare-disease classification, churn, defect
detection. It has been part of the scikit-learn-contrib organization since its
first release and was described in a 2017 JMLR paper by Guillaume Lemaître,
Fernando Nogueira, and Christos Aridas[^1]. The repository was created in
2014[^2].

Its reason to exist is a specific gap in scikit-learn: standard estimators and
transformers never change the number of rows between `fit` and `predict`, so a
resampler — which by definition adds or drops samples — does not fit the
transformer contract. imbalanced-learn defines a parallel `sampler` interface
(`fit_resample`) and ships its own `Pipeline` that applies resampling during
training only. This is the package's defining tension: it looks like
scikit-learn and mostly behaves like it, but the one place it deviates
(sample-count changes) is exactly where misuse causes silent, serious errors.

The library is deliberately narrow. It does not train models, do
cost-sensitive learning, or tune thresholds; it reshapes the training
distribution and hands control back to scikit-learn. Whether resampling is even
the right tool is contested — many practitioners get better calibrated results
from class weighting or threshold tuning — and the project's own documentation
is fairly candid that resampling is one option among several, not a default.

## Getting Started

```bash
pip install -U imbalanced-learn
# or
conda install -c conda-forge imbalanced-learn
```

```python
from imblearn.over_sampling import SMOTE
from imblearn.pipeline import Pipeline          # NOT sklearn.pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score
from sklearn.datasets import make_classification

X, y = make_classification(n_samples=5000, weights=[0.95, 0.05], random_state=0)

# Resampling lives INSIDE the pipeline so it runs per CV fold, not on the
# whole dataset — doing SMOTE before the split leaks synthetic points into
# validation folds and inflates scores.
pipe = Pipeline([
    ("smote", SMOTE(random_state=0)),
    ("clf",   RandomForestClassifier(random_state=0)),
])

scores = cross_val_score(pipe, X, y, scoring="average_precision", cv=5)
print(scores.mean())
```

## Architecture / How It Works

The API splits into four families of estimators, all sharing `fit_resample(X, y)
-> X_res, y_res`:

- **Over-sampling** (`imblearn.over_sampling`) — `RandomOverSampler`, `SMOTE`
  and its variants (`BorderlineSMOTE`, `SVMSMOTE`, `KMeansSMOTE`, `ADASYN`),
  plus `SMOTENC`/`SMOTEN` for categorical and all-categorical features. SMOTE
  interpolates new minority points between existing ones and their
  k-nearest neighbors[^3].
- **Under-sampling** (`imblearn.under_sampling`) — `RandomUnderSampler` plus
  heuristic cleaners (`TomekLinks`, `EditedNearestNeighbours`, `NearMiss`,
  `ClusterCentroids`, `CondensedNearestNeighbour`, `InstanceHardnessThreshold`).
- **Combination** (`imblearn.combine`) — `SMOTEENN`, `SMOTETomek`:
  over-sample then clean.
- **Ensemble** (`imblearn.ensemble`) — `BalancedRandomForestClassifier`,
  `BalancedBaggingClassifier`, `RUSBoostClassifier`, `EasyEnsembleClassifier`.
  These are full classifiers that resample each base estimator's bootstrap.

The critical piece is `imblearn.pipeline.Pipeline`. A sampler exposes
`fit_resample` but no `transform`, so it cannot go in `sklearn.pipeline.Pipeline`.
imbalanced-learn's drop-in replacement invokes resampling only during `fit`;
at `predict`/`transform` time the sampler is a no-op, so the model sees the
real distribution at inference. This is what keeps resampling from corrupting
test data — but only if you actually use it. The estimators are wired to
scikit-learn's tag system and pass its `check_estimator` conformance suite, so
grid search, `cross_val_score`, and column transformers work unchanged.

Optional integrations exist for TensorFlow/Keras batch generators
(`imblearn.keras`, `imblearn.tensorflow`), which yield balanced mini-batches
for deep models rather than materializing a resampled array.

## Production Notes

- **Resampling before the train/test split is the canonical bug.** SMOTE
  applied to the full dataset synthesizes points from samples that later land
  in the validation fold, leaking information and producing scores that do not
  survive deployment. Always resample inside the pipeline/CV loop. This mistake
  is common enough that it shows up in published papers.
- **SMOTE degrades in high dimensions.** Interpolation between nearest
  neighbors assumes local linearity; with many features, sparse categoricals,
  or text/embedding vectors, the synthetic points are often noise. `SMOTENC`
  handles mixed categorical/continuous data but still interpolates continuous
  columns. For truly high-dimensional data, class weighting frequently beats
  resampling.
- **Resampling and probability calibration conflict.** Changing the training
  base rate shifts the model's output probabilities away from the real prior.
  If you need calibrated probabilities (not just ranking), prefer
  `class_weight` or recalibrate afterward.
- **`sampling_strategy` semantics differ by direction.** As a float it means
  the ratio of minority to majority *after* resampling, but the reference class
  is the majority for over-samplers and the minority for under-samplers — read
  the docstring for the specific estimator rather than assuming.
- **Version coupling to scikit-learn is tight.** imbalanced-learn tracks the
  scikit-learn estimator API and enforces a minimum scikit-learn version (1.4+
  on recent releases, Python 3.10+)[^4]. A scikit-learn upgrade can require an
  imbalanced-learn upgrade in the same step; pin both together.
- **Still 0.x after a decade.** The project has never shipped a 1.0, and minor
  releases occasionally change or deprecate estimators (`NeighbourhoodCleaningRule`
  parameters, SMOTE variant defaults). Read release notes before bumping.
- **Determinism needs `random_state`.** Every stochastic sampler takes one;
  omit it and CV folds are not reproducible.

## When to Use / When Not

**Use when:**
- You have a strong minority/majority skew and a model family that lacks a good
  `class_weight` option, or reweighting alone underperforms.
- You want resampling wired correctly into scikit-learn cross-validation
  without hand-rolling the leak-free plumbing.
- You want to A/B several resampling strategies behind one consistent API.

**Avoid when:**
- Your estimator already supports `class_weight="balanced"` or a
  `scale_pos_weight` — try that first; it is simpler and keeps probabilities
  sane.
- You need calibrated probabilities and can instead tune the decision
  threshold on the real distribution.
- Your features are very high-dimensional or dominated by sparse
  categoricals/embeddings, where SMOTE interpolation tends to add noise.

## Alternatives

- scikit-learn/scikit-learn — use its `class_weight` / `sample_weight` and
  threshold tuning instead when reweighting is enough and you want to avoid
  synthetic data entirely.
- analyticalmindsltd/smote_variants — use when you need SMOTE variants that
  imbalanced-learn does not ship (it catalogs dozens of research variants).
- ZhiningLiu1998/imbalanced-ensemble — use when your focus is ensemble methods
  for imbalance with unified evaluation, rather than standalone samplers.
- dmlc/xgboost — use `scale_pos_weight` (or LightGBM's `is_unbalance`) when a
  gradient-boosted model is your estimator and native reweighting suffices.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial repo | 2014-08 | Repository created under what became scikit-learn-contrib[^2]. |
| JMLR paper | 2017 | Lemaître, Nogueira & Aridas, JMLR v18(17)[^1]. |
| 0.x series | 2017–2026 | Ongoing minor releases; no 1.0. SMOTE variants, balanced ensembles, `SMOTENC`/`SMOTEN`, and scikit-learn tag conformance added over time. |
| recent | 2026 | Requires scikit-learn ≥ 1.4.2 and Python ≥ 3.10; endorses SPEC 0 minimum-dependency policy[^4]. |

## References

[^1]: Guillaume Lemaître, Fernando Nogueira, Christos K. Aridas, "Imbalanced-learn: A Python Toolbox to Tackle the Curse of Imbalanced Datasets in Machine Learning", JMLR 18(17):1-5, 2017. http://jmlr.org/papers/v18/16-365
[^2]: GitHub repository metadata, `created_at` 2014-08-16. https://github.com/scikit-learn-contrib/imbalanced-learn
[^3]: N. V. Chawla et al., "SMOTE: Synthetic Minority Over-sampling Technique", JAIR 16:321-357, 2002. https://www.jair.org/index.php/jair/article/view/10302
[^4]: imbalanced-learn README, dependency minimums and SPEC 0 endorsement. https://imbalanced-learn.org/stable/install.html

## Tags

python, machine-learning, imbalanced-data, resampling, smote, scikit-learn, data-science, oversampling, undersampling, classification
