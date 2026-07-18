# arviz-devs/arviz

> The backend-agnostic diagnostics and visualization layer of the Python Bayesian stack — if your sampler emits posterior draws, ArviZ is how you check them.

[GitHub repo](https://github.com/arviz-devs/arviz) ·
[Official website](https://python.arviz.org) ·
[License: Apache-2.0](https://github.com/arviz-devs/arviz/blob/main/LICENSE)

## Overview

ArviZ ("AR-vees") is a Python package for exploratory analysis of Bayesian
models: posterior summaries, MCMC convergence diagnostics, model comparison,
and posterior/prior predictive checking[^1]. Its core design decision is to be
**sampler-agnostic**: PyMC, Stan (CmdStanPy/PyStan), NumPyro, Pyro, emcee, and
TensorFlow Probability all convert their output into ArviZ's `InferenceData`
container, so the same `plot_trace`, `summary`, `loo`, and `rhat` calls work
regardless of which probabilistic programming library produced the draws.

The star count (~1.8k) badly understates its reach. ArviZ is a hard dependency
of PyMC — `pm.sample()` returns an `InferenceData` object directly — and the
de facto standard post-sampling toolkit across the Python PPL ecosystem. It is
a NumFOCUS-affiliated community project with institutional backing from Aalto
University and FCAI, and Aki Vehtari (of Stan and the R-hat/PSIS-LOO papers)
is among its authors, which is why its diagnostics track the current
statistical literature rather than textbook-era defaults[^2].

The defining tension in 2026 is the **1.0 modular split**: the monolithic
`arviz` package is being replaced by `arviz-base` (data + conversion),
`arviz-stats` (diagnostics), and `arviz-plots` (visualization), installable
today via `pip install "arviz[preview]"`[^3]. The main repo now largely
carries documentation and the meta-package (GitHub's language detection
reports "TeX" for this reason), while active development happens in the
sub-package repos. Users face a real migration, with a dedicated guide[^4].

## Getting Started

```bash
pip install arviz                  # stable 0.x API
pip install "arviz[preview]"       # 1.0 modular preview (arviz-base/stats/plots)
```

```python
import arviz as az

# Bundled example: Eight Schools posterior with sample_stats + log_likelihood
idata = az.load_arviz_data("centered_eight")

az.summary(idata, var_names=["mu", "tau"])
# mean, sd, hdi_3%, hdi_97%, mcse, ess_bulk, ess_tail, r_hat per variable

az.plot_trace(idata, var_names=["mu", "tau"])   # convergence at a glance
az.plot_posterior(idata, var_names=["mu"])      # marginal + HDI
az.loo(idata)                                   # PSIS-LOO cross-validation
```

With PyMC there is no conversion step (`pm.sample()` returns `InferenceData`);
with other samplers use `az.from_cmdstanpy(...)`, `az.from_numpyro(...)`, etc.

## Architecture / How It Works

Everything revolves around **`InferenceData`**: a container of labeled xarray
`Dataset` groups following a published schema[^5] — `posterior`, `prior`,
`posterior_predictive`, `log_likelihood`, `sample_stats`, `observed_data`,
and others. Dimensions are conventionally `(chain, draw, *shape)`. Because
groups are xarray Datasets, coordinate-aware indexing, broadcasting, and
out-of-core computation via dask come along for free, and the whole object
serializes to NetCDF or Zarr. The schema is deliberately cross-language:
ArviZ.jl reads and writes the same layout from Julia.

On top of the container sit three concerns, which the 1.0 split makes literal
package boundaries:

- **Conversion** (`arviz-base`): `from_*` adapters per PPL plus
  `convert_to_inference_data` for raw dicts/arrays. This is where coupling to
  each sampler's internals lives, and it is the layer that breaks when a PPL
  changes its output format.
- **Stats** (`arviz-stats`): rank-normalized split R-hat, bulk/tail effective
  sample size, and MCSE per Vehtari et al. 2021[^2]; PSIS-LOO and WAIC for
  model comparison[^6]; HDI computation; `compare()` for ranking models.
- **Plots** (`arviz-plots`): trace, forest, pair, rank, energy, PPC, ESS
  evolution plots. Legacy ArviZ supports matplotlib and bokeh backends; the
  new `arviz-plots` adds plotly and rebuilds plots on a composable
  faceting layer instead of monolithic functions.

## Production Notes

- **Memory is the main operational hazard.** `InferenceData` holds all draws
  in RAM; a `posterior_predictive` or `log_likelihood` group on a model with
  large observed data multiplies size fast (chains × draws × observations ×
  8 bytes). Thin, subset with xarray selection, or use dask-backed arrays.
- **`az.loo` needs a stored pointwise `log_likelihood` group** (PyMC: opt in
  via `pm.sample(idata_kwargs={"log_likelihood": True})`). If PSIS Pareto-k
  diagnostics exceed 0.7, the LOO estimate is unreliable and ArviZ warns —
  those warnings flag influential observations, do not suppress them[^6].
- **The 0.x → 1.0 migration is a real break**, not a rename: plotting
  functions move to `arviz_plots` with different signatures and a
  backend-neutral API. Pin `arviz<1` if you cannot absorb churn yet, and
  budget migration time using the official guide[^4].
- **Version coupling with PyMC.** Because PyMC depends on ArviZ, upgrading
  one can force the other; resolver conflicts in pinned environments are a
  recurring annoyance.
- **Diagnostics defaults have evolved.** R-hat and ESS follow the
  rank-normalized definitions since roughly 2019/2020, so thresholds differ
  from older literature (aim for R-hat < 1.01, not < 1.1)[^2]; comparing
  against numbers from pre-2019 estimators is apples to oranges.
- Maintenance is healthy: pushes within the last month (as of mid-2026),
  ~106 open issues, active multi-repo dev organization.

## When to Use / When Not

**Use when:**
- You run MCMC or variational inference in any Python PPL and need
  convergence checking, posterior visualization, or model comparison.
- You want one storage format (`InferenceData`/NetCDF) for posterior results
  shared across teams, languages (Julia), or archival.
- You are comparing models fit in *different* PPLs — ArviZ is the common ground.

**Avoid when:**
- You work primarily in R — the native Stan-ecosystem tooling is stronger there.
- You need quick corner plots of raw emcee-style chains and nothing else —
  a full InferenceData round-trip is overkill.
- Your workflow is point estimates from scikit-learn-style APIs; ArviZ
  assumes posterior draws exist.

## Alternatives

- stan-dev/bayesplot — the R equivalent for Stan-centric plotting; use it
  when your modeling happens in R/brms/rstanarm.
- stan-dev/posterior — R package for draws containers and the same modern
  R-hat/ESS diagnostics; use for R-side storage and summaries.
- mjskay/tidybayes — use when you want tidyverse/ggplot2-grammar posterior
  workflows instead of prebuilt plot functions.
- dfm/corner.py — use for lightweight corner plots straight from array-shaped
  chains (astronomy/emcee tradition) without the InferenceData machinery.
- arviz-devs/ArviZ.jl — the Julia interface to the same InferenceData schema,
  for Turing.jl/Stan.jl users.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-07 | Repository created; grew out of shared plotting/diagnostic code in the PyMC and Stan communities. |
| 0.1–0.3 | 2018 | First PyPI releases; xarray-backed `InferenceData` design established. |
| 0.x | 2019-01 | JOSS paper (Kumar, Carroll, Hartikainen, Martin)[^1]; adoption as PyMC's return type followed. |
| 0.6+ | 2019–2020 | Bokeh added as a second plotting backend; rank-normalized R-hat/ESS become the defaults[^2]. |
| 1.0 preview | 2025–2026 | Modular split into arviz-base / arviz-stats / arviz-plots with plotly support; second JOSS paper[^3]; `arviz[preview]` install path and migration guide[^4]. |

## References

[^1]: Kumar et al., "ArviZ a unified library for exploratory analysis of Bayesian models in Python", JOSS 2019. https://doi.org/10.21105/joss.01143
[^2]: Vehtari, Gelman, Simpson, Carpenter, Bürkner, "Rank-Normalization, Folding, and Localization: An Improved R-hat for Assessing Convergence of MCMC", Bayesian Analysis 2021. https://doi.org/10.1214/20-BA1221
[^3]: Martin et al., "ArviZ: a modular and flexible library for exploratory analysis of Bayesian models", JOSS 2026. https://doi.org/10.21105/joss.09889
[^4]: ArviZ migration guide (0.x → 1.0). https://python.arviz.org/en/latest/user_guide/migration_guide.html
[^5]: InferenceData schema specification. https://python.arviz.org/en/latest/schema/schema.html
[^6]: Vehtari, Gelman, Gabry, "Practical Bayesian model evaluation using leave-one-out cross-validation and WAIC", Statistics and Computing 2017. https://doi.org/10.1007/s11222-016-9696-4

## Tags

python, bayesian-inference, mcmc, statistics, data-visualization, model-diagnostics, model-comparison, xarray, probabilistic-programming, scientific-computing
