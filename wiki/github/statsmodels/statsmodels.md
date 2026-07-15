# statsmodels/statsmodels

> Inferential statistics and econometrics for Python — the p-values, standard errors, and diagnostic tests scikit-learn deliberately leaves out.

[GitHub repo](https://github.com/statsmodels/statsmodels) ·
[Official website](https://www.statsmodels.org/) ·
[License: BSD-3-Clause](https://github.com/statsmodels/statsmodels/blob/main/LICENSE.txt)

## Overview

statsmodels is a Python package for estimating statistical models and running
hypothesis tests, with a bias toward *inference* rather than *prediction*. It
grew out of the `scipy.stats.models` code (originally by Jonathan Taylor) and
became a standalone project through a 2009 Google Summer of Code effort led by
Skipper Seabold and Josef Perktold, who remain central maintainers[^1]. The
canonical output of a statsmodels fit is not a prediction but a regression
table: coefficients with standard errors, t-statistics, p-values, confidence
intervals, and a battery of goodness-of-fit and specification diagnostics —
the analysis an economist or biostatistician expects from R, Stata, or SAS.

That framing is the whole point and the whole tension. scikit-learn optimizes
for out-of-sample prediction and cross-validation and treats coefficients as
opaque. statsmodels optimizes for understanding *why* a coefficient is what it
is, and will happily hand you a Newey-West or cluster-robust covariance matrix
to defend it. The cost is an older, less uniform codebase: two parallel APIs
(array-based and R-style formula), heavier memory use, no out-of-core support,
and a project that after 15 years has still never shipped a 1.0 release.

It remains the default library for OLS/GLM inference, time-series modeling
(ARIMA, state space, VAR), survival analysis, GEE, and the long tail of
statistical tests that have no clean home elsewhere in the Python stack. For
serious econometrics it is effectively unavoidable.

## Getting Started

```bash
pip install statsmodels          # pulls numpy, scipy, pandas, patsy
# or
conda install -c conda-forge statsmodels
```

```python
import statsmodels.formula.api as smf
import statsmodels.api as sm

# R-style formula API — patsy parses "y ~ x1 + x2"
data = sm.datasets.get_rdataset("mtcars").data
model = smf.ols("mpg ~ wt + hp + C(cyl)", data=data)
results = model.fit()

print(results.summary())          # full regression table with p-values, CIs
print(results.params)             # coefficient Series
print(results.pvalues)            # per-coefficient p-values

# robust (heteroskedasticity-consistent) standard errors
robust = model.fit(cov_type="HC3")
```

The array API (`sm.OLS(endog, exog)`) is the alternative entry point; note it
does **not** add an intercept automatically — you must call `sm.add_constant()`
yourself, a classic first-time footgun.

## Architecture / How It Works

Everything follows a **Model → `.fit()` → Results** triad. A model object holds
the design matrices (named `endog` for the dependent variable, `exog` for
regressors — terminology borrowed from econometrics, not sklearn's `X`/`y`).
Calling `.fit()` runs the estimator and returns a `Results` object whose
`.summary()`, `.params`, `.bse`, `.conf_int()`, and `.predict()` methods expose
the inference. Results objects are the durable artifact; models are mostly a
staging area.

There are two coexisting front ends:

- **`statsmodels.api`** — array-in, array-out. You build the design matrix by
  hand (often via `add_constant`).
- **`statsmodels.formula.api`** — R-style string formulas parsed by **patsy**
  into design matrices. This is where `C(cyl)` categoricals, interactions
  (`x1:x2`), and transforms (`np.log(x)`) come from.

The dependency on **patsy** is a structural fact worth knowing: patsy is
largely in maintenance mode, and the ecosystem's successor, `formulaic`, is not
yet a drop-in replacement here. Formula parsing is therefore both a defining
feature and a long-term maintenance liability.

The time-series side is architecturally distinct. The **state space framework**
(`statsmodels.tsa.statespace`) — SARIMAX, VARMAX, dynamic factor, unobserved
components — is built on a Cython Kalman filter/smoother and is the most
performance-engineered part of the codebase[^2]. It shares little with the
linear-model machinery beyond the Results pattern.

A **sandbox** subpackage ships in every release containing half-finished code
(GMM, panel models, extra distributions) that is explicitly "not production
ready." It is imported by some stable code paths, so it is not purely dead
weight, but you should not build on `statsmodels.sandbox.*` directly.

## Production Notes

**Memory and scale.** statsmodels materializes full design matrices in memory
and offers no streaming/out-of-core path. An OLS on a wide dummy-encoded design
can allocate a dense `exog` far larger than the source frame. There is no
partial-fit; if the data does not fit in RAM, statsmodels is the wrong tool.

**`.summary()` is for humans, not machines.** The pretty regression table is a
`Summary` object built for printing. Extract numbers from `.params`, `.bse`,
`.pvalues`, `.conf_int()`, and `.tvalues` instead of parsing the table text.
`summary().as_csv()` / `.as_latex()` exist but are brittle for programmatic use.

**Convergence on the hard models.** GEE, mixed linear models (`MixedLM`), and
the discrete zero-inflated / negative-binomial families use iterative
optimizers that do not always converge cleanly. Watch for `ConvergenceWarning`,
and expect to tune `method`, `maxiter`, and starting values. A returned Results
object does **not** guarantee the optimizer converged — check
`results.mle_retvals` where available.

**Robust covariance is opt-in.** Default standard errors assume homoskedastic,
independent errors. For real-world data you almost always want `cov_type="HC3"`
(heteroskedasticity) or `cov_type="cluster"` with `cov_kwds={"groups": ...}`.
Forgetting this yields confidently wrong p-values.

**pandas integration is good but not total.** Index alignment and NaN handling
(`missing="drop"`) generally work, but some estimators silently coerce to numpy
and return positional rather than labeled output. Confirm your Results indices.

**Version and dependency coupling.** statsmodels pins fairly tight numpy/scipy/
pandas ranges and periodically drops old Python versions; the 0.14 line
modernized packaging (Meson build, `pyproject.toml`) and moved wheels off the
legacy setup path[^3]. Upgrading statsmodels in a locked scientific stack can
force numpy/scipy upgrades. There is no 1.0 stability promise — APIs in newer
subpackages (especially `tsa`) still shift between minor releases.

**Threading.** Fitting is not designed to be shared across threads; fit
independent models in separate processes rather than sharing a model object.

## When to Use / When Not

**Use when:**
- You need p-values, confidence intervals, and standard errors — not just point
  predictions.
- You are doing econometrics, biostatistics, or any hypothesis-driven analysis
  (OLS/GLM diagnostics, ARIMA/state-space forecasting, survival, GEE, mixed
  models, MANOVA).
- You want R/Stata-style formulas and regression tables inside a Python pipeline.
- You need a specific statistical test (unit root, cointegration, multiple-
  testing correction, normality) that lives nowhere else in Python.

**Avoid when:**
- Your goal is predictive accuracy with cross-validation and hyperparameter
  search — use scikit-learn.
- You need out-of-core / streaming / distributed training on data that exceeds
  RAM.
- You want a fully consistent, single-paradigm API — the array/formula split and
  econometric naming have real friction.
- You need a Bayesian workflow with full posteriors — reach for PyMC.

## Alternatives

- scikit-learn/scikit-learn — use instead when you care about prediction and
  cross-validation and treat coefficients as opaque.
- scipy/scipy — use `scipy.stats` for standalone tests and distributions when
  you don't need a full modeling/inference layer.
- bashtage/linearmodels — use for panel data, instrumental variables (2SLS/GMM),
  and asset-pricing models that statsmodels covers weakly or only in sandbox.
- pymc-devs/pymc — use when you want Bayesian inference with full posterior
  distributions rather than frequentist point estimates.
- raphaelvallat/pingouin — use for a friendlier, pandas-native API over common
  statistical tests (ANOVA, correlations, effect sizes).

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2009 | GSoC spin-out of `scipy.stats.models` under Seabold & Perktold[^1]. |
| 0.4 | 2012 | First widely-used standalone release; formula/patsy support maturing. |
| 0.6 | 2014 | GLM/GEE expansion, mixed models, improved pandas integration. |
| 0.8 | 2016 | State space time-series framework (SARIMAX, VARMAX) stabilized[^2]. |
| 0.9 | 2018 | Markov switching, dynamic factor models, survival additions. |
| 0.12 | 2020 | Modern ARIMA API; older `tsa.ARMA`/`ARIMA` classes deprecated. |
| 0.13 | 2021 | Continued state-space and count-model work. |
| 0.14 | 2023 | Meson/`pyproject.toml` build, packaging modernization, Python drops[^3]. |

## References

[^1]: statsmodels, "About statsmodels" and project history — the package
originated from `scipy.stats.models` and matured through Google Summer of Code
2009. https://www.statsmodels.org/stable/index.html
[^2]: Chad Fulton, "Estimating time series models by state space methods in
Python: statsmodels" — describes the Cython Kalman filter/smoother framework.
https://www.statsmodels.org/stable/statespace.html
[^3]: statsmodels release notes (0.14 series), covering the Meson build
migration and supported-Python changes. https://www.statsmodels.org/stable/release/
[^4]: Source repository and issue tracker.
https://github.com/statsmodels/statsmodels

## Tags

python, statistics, econometrics, regression, time-series, hypothesis-testing, generalized-linear-models, data-science, inference, forecasting, scientific-computing
