# shap/shap

> Game-theoretic feature attribution — Shapley values as a unified, model-agnostic explanation for any ML model, with fast exact algorithms for trees.

[GitHub repo](https://github.com/shap/shap) ·
[Official website](https://shap.readthedocs.io) ·
[License: MIT](https://github.com/shap/shap/blob/master/LICENSE)

## Overview

SHAP (SHapley Additive exPlanations) computes, for a single prediction, how much each input feature pushed the model output away from a baseline. It grounds this in Shapley values from cooperative game theory: treat features as "players", the prediction as the "payout", and distribute credit by the unique allocation that satisfies local accuracy, missingness, and consistency[^1]. The original 2017 NeurIPS paper by Scott Lundberg and Su-In Lee unified several earlier attribution methods (LIME, DeepLIFT, layer-wise relevance propagation, Shapley sampling) as special cases of one additive framework[^1].

The defining tension is exactness versus tractability. Exact Shapley values require evaluating the model over every subset of features — exponential in feature count. SHAP's value is not the math (Shapley values predate it by decades) but the collection of practical estimators: an exact polynomial-time algorithm for tree ensembles (Tree SHAP)[^2], gradient-based approximations for deep nets, and a model-agnostic sampling estimator (Kernel SHAP) for everything else. Which estimator you use, and how you configure its background distribution, dominates both runtime and what the numbers actually mean.

SHAP is the de facto standard for tabular model explanation in industry and a common component of model-risk documentation. It is a library for interrogating trained models, not for training them, and its explanations describe the model's behavior — not causal structure in the underlying data, a distinction that is routinely mishandled by users. The repository was originally `slundberg/shap` under Lundberg's personal account and has since moved to a community-maintained `shap` organization[^3].

## Getting Started

```bash
pip install shap
# or
conda install -c conda-forge shap
```

```python
import xgboost, shap

X, y = shap.datasets.california()
model = xgboost.XGBRegressor().fit(X, y)

# unified API: shap.Explainer auto-dispatches to the right algorithm
explainer = shap.Explainer(model)          # -> TreeExplainer for XGBoost
shap_values = explainer(X)                  # Explanation object, one row per sample

shap.plots.waterfall(shap_values[0])        # local: one prediction
shap.plots.beeswarm(shap_values)            # global: all features, all samples
```

The `Explanation` object carries `.values`, `.base_values`, and `.data` together so plots stay aligned. For non-tree models pass a prediction function and a background sample: `shap.Explainer(model.predict, background)`.

## Architecture / How It Works

SHAP is a family of explainers unified behind `shap.Explainer`, which inspects the model and picks an algorithm. The important ones differ enormously in cost and assumptions:

- **TreeExplainer (Tree SHAP)** — exact, polynomial-time SHAP values for tree ensembles (XGBoost, LightGBM, CatBoost, scikit-learn, pyspark) via a C++ extension. This is the reason SHAP is practical at scale; it is orders of magnitude faster than the model-agnostic path. Supports pairwise `shap_interaction_values`[^2].
- **KernelExplainer (Kernel SHAP)** — model-agnostic. Fits a weighted local linear regression over sampled feature coalitions to approximate Shapley values for any function. Correct in principle for anything, but slow (many model evaluations per explained row) and only approximate.
- **DeepExplainer / GradientExplainer** — approximations for deep nets built on connections to DeepLIFT and Integrated Gradients respectively. Use a background distribution rather than a single reference.
- **LinearExplainer**, **PermutationExplainer**, **PartitionExplainer / Exact** — closed-form for linear models, and coalition-structured estimators (Partition SHAP uses a hierarchy of feature groups, which is what makes explaining transformer text feasible with few evaluations).

Every estimator depends on a **background dataset** (a "masker" in the modern API) that defines what "a feature is absent" means. SHAP replaces a masked feature by sampling from this background, so the baseline (`base_values`) is the mean model output over it. Change the background, change every SHAP value. Tree SHAP additionally offers two perturbation modes: `tree_path_dependent` (follows the tree's own coverage, no background needed, but entangles correlated features) and `interventional` (requires a background, breaks feature correlations per Janzing et al.'s causal reading). These give different numbers on correlated data and the choice is a genuine modeling decision, not a default to ignore.

The visualization layer (`shap.plots.*`: waterfall, beeswarm, bar, scatter/dependence, force, heatmap) is a large part of the library's real-world value and mixes matplotlib output with JavaScript force plots that need `shap.initjs()` in notebooks.

## Production Notes

- **Background choice is the top footgun.** A large background makes Kernel/Deep explainers unusably slow; a badly chosen one produces misleading baselines. Summarize with `shap.sample(X, 100)` or `shap.kmeans` and understand that base value moves with it.
- **KernelExplainer does not scale.** Cost grows with samples × background size × `nsamples`. For anything beyond a few hundred rows of a black-box model it is often impractical; prefer a model-specific explainer or `PermutationExplainer` where possible.
- **Deep-learning explainers are version-fragile.** DeepExplainer/GradientExplainer have a long history of breaking against new TensorFlow/Keras releases and have only preliminary PyTorch support; pin versions and expect friction. This is one of the most common sources of open issues.
- **SHAP explains the model, not the world.** Attributions are not causal effects. With correlated features, credit splits among the correlated group in ways that mislead if read as data-level importance; `tree_path_dependent` mode can assign nonzero value to features the model never split on.
- **Interaction values are O(features²) per row** and only exact for trees — expensive and memory-heavy on wide datasets.
- **Heavy install.** Ships a compiled C++ extension; GPU Tree SHAP requires building from source with `SHAP_ENABLE_CUDA=1` and the CUDA toolkit present.
- **API churn.** The library is still pre-1.0. The legacy API (`explainer.shap_values(X)` returning raw arrays, `shap.force_plot`, per-explainer classes) and the newer callable/`Explanation`/`shap.plots.*` API coexist; older tutorials frequently use signatures that have shifted. The project follows SPEC 0 for minimum dependency versions[^4].
- **Maintenance cadence.** After the original author's involvement wound down, the project became community-maintained; issue response can be slow and the tracker carries a large backlog (~1,000 open issues), though releases continue.

## When to Use / When Not

**Use when:**
- You need per-prediction feature attributions for a tree ensemble — TreeExplainer is fast, exact, and the industry-standard choice.
- You require defensible, theoretically grounded explanations for model-risk or regulatory documentation.
- You want consistent local and global views (beeswarm/bar) from the same attribution basis.

**Avoid / be cautious when:**
- You need causal insight about the data rather than the model — SHAP answers a different question.
- You're explaining a slow black-box model over large data with KernelExplainer — the runtime is often prohibitive; consider a surrogate or a model-specific method.
- You need real-time explanations at inference latency — most estimators are batch/offline tools.
- Your features are strongly correlated and stakeholders will read attributions as importance — the caveats are easy to get wrong.

## Alternatives

- marcotcr/lime — local surrogate explanations; simpler and faster than Kernel SHAP but without the consistency guarantees.
- interpretml/interpret — Microsoft's InterpretML; use when you want glass-box models (EBMs) that are inherently interpretable instead of post-hoc attribution.
- pytorch/captum — use for PyTorch-native attribution (Integrated Gradients, DeepLIFT, and a Shapley sampling implementation) tightly integrated with autograd.
- SeldonIO/alibi — broader explanation toolbox (anchors, counterfactuals, ALE) when you want alternatives to attribution.
- Trusted-AI/AIX360 — IBM's explainability suite when you need a survey of methods under one API.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-11-22 | Repository created (`slundberg/shap`). |
| — | 2017-12 | NeurIPS paper: "A Unified Approach to Interpreting Model Predictions"[^1]. |
| — | 2020-01 | Tree SHAP published in Nature Machine Intelligence[^2]. |
| 0.36+ | ~2020 | Unified callable `Explainer` API, `Explanation` objects, `shap.plots.*`, maskers. |
| — | — | Repository moved to the community-maintained `shap` organization[^3]. |
| 0.x | ongoing | Still pre-1.0 as of 2026; SPEC 0 dependency policy, continued releases[^4]. |

## References

[^1]: Lundberg, S. & Lee, S.-I. "A Unified Approach to Interpreting Model Predictions." NeurIPS 2017. https://arxiv.org/abs/1705.07874
[^2]: Lundberg, S. et al. "From local explanations to global understanding with explainable AI for trees." Nature Machine Intelligence, 2020. https://www.nature.com/articles/s42256-019-0138-9
[^3]: SHAP repository (transferred to the `shap` organization). https://github.com/shap/shap
[^4]: SPEC 0 — Minimum Supported Dependencies. https://scientific-python.org/specs/spec-0000/

## Tags

python, machine-learning, explainability, interpretability, shapley-values, feature-attribution, xai, game-theory, model-agnostic, tree-ensembles, deep-learning
