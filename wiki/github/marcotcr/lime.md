# marcotcr/lime

> Explain individual predictions of any black-box classifier by fitting a sparse linear surrogate in the neighborhood of the instance.

[GitHub repo](https://github.com/marcotcr/lime) ·
[License: BSD-2-Clause](https://github.com/marcotcr/lime/blob/master/LICENSE)

## Overview

LIME (Local Interpretable Model-agnostic Explanations) is the reference
implementation of the technique from Ribeiro, Singh, and Guestrin's 2016 KDD
paper "Why Should I Trust You?"[^1]. It is one of the two libraries — SHAP being
the other — that defined the post-hoc explainability tooling era. The premise is
model-agnostic: you hand LIME a `predict_proba`-style function and one instance,
and it returns a short list of features weighted by their local contribution to
that single prediction.

The method treats the model as a black box. It perturbs the instance to explain,
queries the model on those perturbed samples, weights each sample by its
proximity to the original, and fits a sparse linear model on the weighted set.
The linear model's coefficients are the explanation. The key assumption is that
even a globally nonlinear decision surface is approximately linear in a small
neighborhood — so the explanation is only ever *locally* faithful, never a
statement about the model as a whole.

GitHub reports the repo's primary language as JavaScript; that is the vendored
d3 visualization bundle. The library itself is Python (scikit-learn, numpy,
scipy). As of this writing the repo has ~12k stars but the last commit landed in
mid-2024 and there has been no new PyPI release in years — LIME is stable and
widely cited but effectively in maintenance-only mode, with active XAI
development having moved to SHAP, Captum, and InterpretML.

## Getting Started

```sh
pip install lime
```

```python
from lime.lime_tabular import LimeTabularExplainer

# training_data is a numpy array; the explainer needs it to compute
# per-feature statistics and (by default) to discretize continuous columns.
explainer = LimeTabularExplainer(
    training_data=X_train,
    feature_names=feature_names,
    class_names=class_names,
    discretize_continuous=True,
    random_state=42,          # perturbation is random; pin it for reproducibility
)

exp = explainer.explain_instance(
    data_row=X_test[0],
    predict_fn=model.predict_proba,   # must return a (n_samples, n_classes) array
    num_features=10,
    num_samples=5000,
)

print(exp.as_list())          # [(feature_description, weight), ...]
# exp.show_in_notebook()      # renders the HTML explanation in Jupyter
```

For text, `lime.lime_text.LimeTextExplainer` perturbs by removing words from the
document; for images, `lime.lime_image.LimeImageExplainer` toggles superpixels
produced by a segmentation algorithm (quickshift by default).

## Architecture / How It Works

LIME is a thin core (`lime_base.LimeBase`) wrapped by three domain-specific
explainers:

- **`lime_text`** — represents a document as a bag of words. Perturbation =
  randomly dropping words; features are word-presence indicators. Explanations
  map directly onto the original tokens for highlighting.
- **`lime_tabular`** — the most-used and most-caveated module. Continuous
  features are discretized (quartile / decile / entropy) so perturbation samples
  land on training-data-plausible values. Categorical features are sampled from
  their observed frequency. It needs the training set (or its summary stats) to
  do this.
- **`lime_image`** — segments the image into superpixels, then perturbs by
  graying out random subsets of them. Explanation weights attach to superpixels,
  not pixels.

The shared core does the actual work: build a perturbed neighborhood, weight
each sample by an exponential kernel over cosine/euclidean distance
(default kernel width `sqrt(n_features) * 0.75`), select features
(`forward_selection`, `lasso_path`, `highest_weights`, or `auto`), then fit a
weighted `Ridge` regression as the surrogate. The returned `Explanation` object
carries the surrogate's coefficients, intercept, and an R² local-fidelity score.

**SP-LIME** (`submodular_pick.SubmodularPick`) sits on top: it runs LIME over
many instances and greedily selects a small, non-redundant representative set via
submodular optimization, so a human can approximate a *global* understanding
from a handful of local explanations.

## Production Notes

- **Explanations are not deterministic.** Because the neighborhood is sampled
  randomly, two runs on the same instance can yield different — occasionally
  contradictory — weights. Always pass `random_state`, and treat single-run
  explanations with suspicion. This instability is the most-cited criticism of
  LIME in the literature[^2].
- **Kernel width is an unprincipled hyperparameter.** The default
  (`sqrt(n_features) * 0.75`) is a heuristic with no theoretical basis, and the
  explanation can flip qualitatively as you change it. There is no built-in way
  to choose it well.
- **Tabular perturbation assumes feature independence.** Features are perturbed
  independently, which can generate out-of-distribution samples for correlated
  features and mislead the surrogate. This is the standard argument for
  preferring Shapley-based methods on tabular data.
- **Cost scales with `num_samples`.** Each explanation calls your model
  `num_samples` times (default 5000 for text/tabular, 1000 for images). For a
  slow model or an image explainer with fine segmentation, a single explanation
  can take seconds to minutes.
- **Image explanations are segmentation-sensitive.** The superpixel algorithm
  and its parameters change the explanation as much as the model does; a bad
  segmentation produces meaningless attributions.
- **Local only.** A LIME explanation says nothing about the model outside the
  sampled neighborhood. Do not present it as a global feature-importance ranking
  (that is what SP-LIME, or a different method, is for).
- **Maintenance risk.** No recent releases; some dependencies (older numpy /
  scikit-learn API assumptions) can surface deprecation warnings on modern
  stacks. Pin versions in production.

## When to Use / When Not

**Use when:**
- You need a fast, model-agnostic sanity check on why one prediction came out
  the way it did, across text, tabular, or image inputs.
- The model is a true black box (no gradients, remote API) and you only have
  `predict_proba`.
- You want human-legible, sparse explanations for a small number of instances.

**Avoid when:**
- You need stable, reproducible attributions for audit or compliance — the
  sampling variance is a liability.
- Your features are strongly correlated (tabular) — independent perturbation
  misleads.
- You have gradient access to a deep model — gradient-based attribution
  (Integrated Gradients via Captum) is faster and more faithful.
- You want theoretically consistent, additive attributions — use Shapley values.

## Alternatives

- slundberg/shap — Shapley-value attributions; KernelSHAP generalizes LIME with
  a principled weighting. Use when you want consistency and both local + global.
- pytorch/captum — gradient-based attribution for PyTorch models. Use when you
  have a differentiable network and want faster, deterministic saliency.
- interpretml/interpret — Microsoft's glassbox (EBM) + blackbox toolkit. Use
  when you can adopt an inherently interpretable model instead of explaining one.
- marcotcr/anchor — the same author's follow-up; rule-based, high-precision
  "anchors" instead of linear weights. Use when you want if-then explanations.
- Trusted-AI/AIX360 — IBM's broad explainability suite. Use when you need many
  explanation methods behind one API.

## History

| Version | Date | Notes |
|---------|------|-------|
| paper | 2016-02 | "Why Should I Trust You?" published (arXiv 1602.04938)[^1]. |
| initial | 2016-03 | Repository created; text and tabular explainers[^3]. |
| 0.1.1.37 | — | Last release with Python 2 support[^4]. |
| 0.2.0 | — | Python 2 dropped; image explainer + SP-LIME matured[^4]. |
| 0.2.0.1 | ~2020 | Latest PyPI release; repo now maintenance-only. |

## References

[^1]: Ribeiro, Singh, Guestrin, "Why Should I Trust You?: Explaining the
Predictions of Any Classifier" — KDD 2016. https://arxiv.org/abs/1602.04938
[^2]: Alvarez-Melis and Jaakkola, "On the Robustness of Interpretability
Methods" — 2018, on the instability of LIME/SHAP explanations.
https://arxiv.org/abs/1806.08049
[^3]: Repository metadata, created 2016-03-15.
https://github.com/marcotcr/lime
[^4]: README installation note: "We dropped python2 support in `0.2.0`,
`0.1.1.37` was the last version before that."
https://github.com/marcotcr/lime#installation

## Tags

python, explainability, interpretability, xai, machine-learning, model-agnostic, feature-attribution, lime, scikit-learn, research
