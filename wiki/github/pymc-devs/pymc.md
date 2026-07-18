# pymc-devs/pymc

> Bayesian modeling and probabilistic programming in Python — write a model as
> Python code, get a posterior via NUTS without hand-deriving gradients.

[GitHub repo](https://github.com/pymc-devs/pymc) ·
[Official website](https://www.pymc.io) ·
[License: Apache-2.0](https://github.com/pymc-devs/pymc/blob/main/LICENSE)

## Overview

PyMC is a probabilistic programming library: you declare random variables and
their dependencies inside a `with pm.Model()` context, and the library builds a
symbolic log-probability graph, differentiates it automatically, and samples
the posterior with gradient-based MCMC — primarily the No-U-Turn Sampler
(NUTS)[^1] — or approximates it with variational inference (ADVI)[^2]. Its
users are mostly statisticians, data scientists, and researchers doing applied
Bayesian work: hierarchical regressions, A/B testing, epidemiology,
marketing-mix models, sports analytics.

The project is one of the oldest living pieces of the scientific-Python stack
(repo created 2009; the lineage goes back further) and its defining tension is
its compute backend. PyMC3 was built on Theano; when Theano was abandoned, the
PyMC team maintained the fork themselves — first as Aesara, then, after a
governance split in late 2022, as **PyTensor**[^3]. Model code stays the same,
but every model you write compiles through this tensor library, and the
backend story (C, JAX, Numba) has changed underneath users several times. PyMC
6.0 (May 2026) made Numba the default compilation backend via PyTensor 3.0 and
made nutpie, a Rust NUTS implementation, the default sampler when
installed[^4].

At ~9.7k stars, 2.3k forks, and pushes within days of any given moment, it is
actively maintained by a NumFOCUS-sponsored core team; 478 open issues reflect
a large research-user population more than neglect.

## Getting Started

```bash
pip install pymc            # safely pip-installable since 6.0
pip install "pymc[nutpie]"  # optional: faster Rust NUTS sampler
```

```python
import pymc as pm

observed = [1.2, 0.8, 1.5, 0.9, 1.1]

with pm.Model() as model:
    mu = pm.Normal("mu", mu=0, sigma=1)
    sigma = pm.HalfNormal("sigma", sigma=1)
    y = pm.Normal("y", mu=mu, sigma=sigma, observed=observed)

    idata = pm.sample()          # NUTS, 4 chains, tuning included

print(pm.stats.summary(idata, var_names=["mu", "sigma"]))
```

`pm.sample()` returns results in ArviZ's data structures (an
`xarray.DataTree` since ArviZ 1.0) for diagnostics and plotting.

## Architecture / How It Works

The pipeline from model code to posterior:

1. **Model context** — `with pm.Model():` intercepts distribution calls.
   `pm.Normal("mu", ...)` registers a random variable as a node in a PyTensor
   computation graph; `observed=` marks likelihood terms.
2. **Logp graph** — PyMC transforms the generative graph into a
   log-probability graph, automatically applying variable transforms
   (e.g. log-transform for positive-support parameters like `HalfNormal`) so
   samplers operate on unconstrained space.
3. **Compilation** — PyTensor rewrites and compiles the graph. As of PyTensor
   3.0 the default backend is Numba; C (`linker = "cvm"`) and JAX remain
   options[^4]. Compilation happens at first call, not at model definition.
4. **Sampling** — NUTS with joint adaptation of step size and mass matrix
   during tuning. Discrete variables cannot take gradients, so PyMC falls
   back to compound steps (Metropolis for discrete + NUTS for continuous),
   which mixes far worse than a marginalized formulation.
5. **External samplers** — `nuts_sampler="nutpie" | "numpyro" | "blackjax"`
   hands the compiled logp to faster implementations. nutpie (Rust) is the
   default when installed since 6.0, with a different tuning default (400
   steps vs PyMC's 1000)[^4].

The coupling story is the thing to understand: PyMC is effectively the
front-end of a three-project stack it also maintains — PyTensor (compute),
nutpie (sampling), plus ArviZ (diagnostics, shared governance). Upgrades to
any of them surface in PyMC's behavior, which is why major PyMC releases are
ecosystem releases[^4].

`pm.do()` and `pm.observe()` implement graph surgery for causal
interventions and conditioning — a distinctive feature; most PPLs make you
rewrite the model instead.

## Production Notes

- **Compile-before-sample latency.** Every model pays a graph-compilation
  cost on first execution — seconds to minutes for large models. For
  iterate-heavy workflows this dominates wall time more than sampling does.
  Numba's JIT warm-up in 6.x shifts where the cost lands but does not remove
  it.
- **Divergences are a modeling problem, not a knob problem.** Hierarchical
  models routinely emit `divergences` warnings; the fix is reparameterization
  (non-centered parameterization), not raising `target_accept` forever.
  Ignoring divergences yields silently biased posteriors.
- **ArviZ 1.0 broke silently-load-bearing defaults.** With PyMC 6.0,
  `InferenceData` became `xarray.DataTree`, `plot_trace` was replaced, and
  the default credible interval changed from 94% HDI to 89% equal-tailed[^4].
  Reported intervals change without any code change — audit downstream
  reports when migrating 5.x → 6.x.
- **`sample_posterior_predictive` semantics changed in 6.0**: "volatile"
  variables now warn instead of being silently resampled; you must assign
  them to `sample_vars` or `freeze_vars` explicitly[^4]. Code ported from
  5.x can produce warnings that mask real semantic differences.
- **Multiprocessing quirks.** Parallel chains are separate processes; on
  Windows and macOS (spawn start method) models defined in notebooks or
  closures can hit pickling errors. `cores=1` or nutpie sidesteps this.
- **Scaling ceiling.** NUTS is practical to roughly 10^4–10^5 parameters and
  full-data likelihoods; it is not minibatch-friendly. Beyond that, ADVI/
  minibatch ADVI trades accuracy for speed, or move to NumPyro on GPU.
- **Pin the stack.** PyMC, PyTensor, ArviZ, and nutpie versions are
  co-released and cross-constrained; letting a resolver float one of them is
  a recurring source of breakage in deployed environments.

## When to Use / When Not

**Use when:**
- You need full Bayesian inference with uncertainty quantification —
  hierarchical/multilevel models, small-to-medium data, decision-making
  under uncertainty.
- You want the model readable as Python by non-specialists, with the
  largest tutorial/book ecosystem of any Python PPL.
- You want causal-inference workflows (`pm.do`, `pm.observe`) integrated
  with sampling.

**Avoid when:**
- You need maximum sampling throughput on GPUs at scale — NumPyro or
  BlackJAX on JAX is faster for large models.
- Your model is mostly discrete/combinatorial — gradient-based MCMC cannot
  help, and compound Metropolis steps mix poorly.
- You just need point estimates or regularized regression — scikit-learn or
  statsmodels is simpler and orders of magnitude faster.
- You need deep-learning-integrated inference (Bayesian NNs at scale) —
  Pyro/NumPyro sit closer to that stack.

## Alternatives

- stan-dev/stan — use instead when you want the most battle-tested NUTS
  implementation and a model language shared across R/Python/Julia, at the
  cost of writing Stan code and a C++ toolchain.
- pyro-ppl/numpyro — use instead when you need JAX-native speed and GPU/TPU
  sampling for large models.
- pyro-ppl/pyro — use instead for deep probabilistic models and SVI on
  PyTorch.
- tensorflow/probability — use instead when already committed to the
  TensorFlow ecosystem; lower-level, more assembly required.
- blackjax-devs/blackjax — use instead when you have your own logp function
  and only want composable JAX samplers, no modeling layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| PyMC (orig.) | 2003 | Started by Chris Fonnesbeck; Fortran-backed MCMC[^5]. |
| PyMC3 3.0 | 2017-01 | Rewrite on Theano; NUTS + ADVI[^6]. |
| 4.0 | 2022-06 | Renamed PyMC3 → PyMC; Theano fork (Aesara) backend[^3]. |
| 5.0 | 2022-12 | Switch to PyTensor after fork from Aesara[^3]. |
| 5.x series | 2023–2026 | Dims module, external samplers, `pm.do`/`pm.observe`. |
| 6.0.0 | 2026-05-13 | PyTensor 3.0 (Numba default backend), nutpie default sampler, ArviZ 1.0 / DataTree[^4]. |
| 6.1.0 | 2026-07-07 | Latest release at time of writing. |

## References

[^1]: Hoffman & Gelman, "The No-U-Turn Sampler" — JMLR 15, 2014. http://www.jmlr.org/papers/v15/hoffman14a.html
[^2]: Kucukelbir et al., "Automatic Differentiation Variational Inference" — JMLR 18, 2017. http://www.jmlr.org/papers/v18/16-107.html
[^3]: PyMC docs, "About PyMC / history of the backend" (Theano → Aesara → PyTensor). https://www.pymc.io/projects/docs/en/stable/learn.html
[^4]: PyMC v6.0.0 release notes and ecosystem announcement — 2026-05-13. https://github.com/pymc-devs/pymc/releases/tag/v6.0.0 and https://www.pymc.io/blog/pymc_v6_ecosystem_updates.html
[^5]: Salvatier, Wiecki & Fonnesbeck, "Probabilistic programming in Python using PyMC3" — PeerJ CS, 2016. https://doi.org/10.7717/peerj-cs.55
[^6]: PyMC3 3.0 release — 2017-01. https://github.com/pymc-devs/pymc/releases/tag/v3.0

## Tags

python, bayesian-inference, probabilistic-programming, mcmc, variational-inference, statistics, pytensor, scientific-computing, data-science, library
