# JuliaStats/Distributions.jl

> The canonical Julia package for probability distributions — the de facto interface the entire Julia statistics and probabilistic-programming ecosystem dispatches on.

[GitHub repo](https://github.com/JuliaStats/Distributions.jl) ·
[Documentation](https://juliastats.org/Distributions.jl/stable/) ·
[License: MIT](https://github.com/JuliaStats/Distributions.jl/blob/master/LICENSE.md)[^1]

## Overview

Distributions.jl provides types and functions for probability distributions: densities and log-densities, CDFs and quantiles, moments (mean, variance, skewness, kurtosis), entropy, MGFs/characteristic functions, random sampling, and maximum-likelihood fitting. The repo dates to November 2012 — six years before Julia 1.0 — and is maintained by the JuliaStats organization, with a peer-reviewed description published in the Journal of Statistical Software in 2021[^2].

Its real significance is not the ~100 built-in distributions but the *interface*: `pdf`, `logpdf`, `cdf`, `quantile`, `rand`, `mean`, `var`, and friends are generic functions that downstream packages extend and dispatch on. Turing.jl, and most Bayesian, econometrics, and simulation code in Julia, accept "anything that implements the Distributions API". That makes it load-bearing infrastructure comparable to `scipy.stats` in Python.

The defining tension follows from that position: the package has never reached 1.0. It has sat on the v0.25 series since May 2021, accumulating 129 patch releases by mid-2026[^3] — under Julia's 0.x semver convention a minor bump is breaking, and a break here ripples through hundreds of dependents, so features ship as patches while breaking redesigns stall. The tracker (475 open issues + PRs, July 2026) shows more surface area than review bandwidth.

## Getting Started

```julia
pkg> add Distributions
```

```julia
using Distributions, Random

d = Normal(2.0, 0.5)              # immutable struct holding μ, σ
mean(d), std(d)                   # (2.0, 0.5)
logpdf(d, 2.3)                    # prefer logpdf over log(pdf(...))
quantile(d, 0.95)                 # inverse CDF

x = rand(d, 10_000)               # vectorized sampling
fit_mle(Normal, x)                # MLE via sufficient statistics

t  = truncated(Normal(0, 1), -2, 2)      # wrapper combinator
mv = MvNormal([0.0, 0.0], [1.0 0.5; 0.5 1.0])
rand(mv, 100)                     # 2×100 matrix, columns are draws
```

## Architecture / How It Works

Everything hangs off one abstract type, `Distribution{F<:VariateForm, S<:ValueSupport}`, a two-axis classification: variate form (`Univariate`, `Multivariate`, `Matrixvariate`) × value support (`Discrete`, `Continuous`), with aliases like `ContinuousUnivariateDistribution`. Concrete distributions are small immutable structs storing parameters; all behavior lives in multiply-dispatched methods (`pdf(d, x)`, `rand(rng, d)`, …). Adding a distribution means defining a struct plus a handful of methods — the docs specify the minimal method set, and the test suite runs `Test.detect_ambiguities` because a hub package with this much dispatch surface breeds method ambiguities.

Composition is done with wrapper types rather than new densities: `truncated(d, l, u)`, `censored`, `product_distribution`, `MixtureModel`, and `convolve` produce new distribution objects that forward to the wrapped ones. `sampler(d)` returns a pre-computed sampler for amortizing per-draw setup cost across many samples.

The coupling story is the JuliaStats stack itself. The generic function names are owned by StatsAPI/StatsBase and re-exported here. Numeric kernels for many classical CDFs and quantiles come from StatsFuns.jl, which historically delegates several to Rmath-julia — a C library extracted from R[^4]. Multivariate normals build on PDMats.jl's positive-definite matrix types (`PDMat`, `PDiagMat`, `ScalMat`), so a dense `MvNormal` Cholesky-factorizes its covariance once at construction and reuses it for every `logpdf` and `rand` call[^5]. Special functions come from SpecialFunctions.jl. Conjugate priors were split out early into the separate ConjugatePriors.jl package.

## Production Notes

**The Rmath boundary is the classic footgun.** Code paths that bottom out in the Rmath C library are Float64-only: they silently defeat automatic differentiation (ForwardDiff dual numbers), arbitrary-precision `BigFloat`, and GPU number types for the affected `cdf`/`quantile`/`logpdf` methods. This is why TuringLang maintains DistributionsAD.jl, a patch layer that re-derives differentiable implementations[^6]. If you are doing gradient-based inference, check whether your distribution's methods are pure Julia before assuming AD works. Coverage has improved over the 0.25 series, but unevenly.

**Semver is nonstandard by necessity.** Pin `Distributions = "0.25"` in `[compat]` and expect *features* to arrive in patch releases — the project ships additions as 0.25.x precisely to avoid ecosystem-wide breakage. The flip side: a future 0.26 will be a coordinated ecosystem event, not a routine bump.

**Performance is construction-time vs call-time.** Distribution structs are cheap, but `MvNormal` with a dense covariance pays a Cholesky at construction — hoist it out of loops, and use `PDiagMat`/`ScalMat` when covariance structure allows. For hot sampling loops, use `sampler(d)` and `rand!` into preallocated buffers rather than repeated `rand(d, n)`.

**Smaller footguns.** Constructors validate parameters (`Normal(0, -1)` throws); `check_args=false` skips this in hot paths at your own risk. Parameters promote to a common type (`Normal(0, 1)` is `Normal{Float64}`) — relevant for `Float32`-end-to-end pipelines. Custom distributions must implement the documented minimal method set, and method ambiguities against wrapper types (`truncated`, product distributions) are the common failure mode reported by extension authors.

## When to Use / When Not

**Use when:**
- You are writing statistical, simulation, or Bayesian code in Julia — this is the standard vocabulary, and interop with Turing.jl and the JuliaStats stack assumes it.
- You need MLE fitting, truncation/censoring, or mixtures with a uniform API.
- You are authoring a package that should accept "any distribution".

**Avoid when:**
- You need guaranteed AD-compatibility across every distribution — audit the Rmath-backed paths first, or consider a log-density-first design.
- You work with unnormalized densities or measure-theoretic constructions; the normalized-distribution abstraction is a poor fit.
- You are not in Julia — this package is the ecosystem, not a portable spec.

## Alternatives

- JuliaMath/MeasureTheory.jl — use instead when you need unnormalized measures, log-density-first design, and fewer AD surprises (Julia).
- scipy/scipy — `scipy.stats`; use when your stack is Python/NumPy and you need battle-tested frequentist routines rather than composable types.
- pytorch/pytorch — `torch.distributions`; use for batched, reparameterized sampling inside deep-learning/variational-inference training loops.
- tensorflow/probability — use for TF/JAX-substrate probabilistic layers and bijectors at scale.
- stan-dev/math — use when you want C++ density/gradient kernels with built-in autodiff backing HMC, independent of a host language ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| (repo) | 2012-11 | Created by Doug Bates, John Myles White, et al.; among the earliest JuliaStats packages[^1]. |
| v0.16.0 | 2018-07 | Release series spanning the Julia 0.7 → 1.0 transition. |
| v0.20.0 | 2019-05 | Pre-JSS-paper consolidation of the API[^3]. |
| v0.24.0 | 2020-10 | Last minor series before 0.25[^3]. |
| v0.25.0 | 2021-05 | Start of the current series; JSS paper published the same year[^2][^3]. |
| v0.25.129 | 2026-06 | Latest release — 129 patches on one minor series over five years[^3]. |

## References

[^1]: Distributions.jl LICENSE.md (MIT; GitHub reports the license as non-standard because the file also covers bundled components). https://github.com/JuliaStats/Distributions.jl/blob/master/LICENSE.md
[^2]: Besançon et al., "Distributions.jl: Definition and Modeling of Probability Distributions in the JuliaStats Ecosystem", Journal of Statistical Software 98(16), 2021. https://doi.org/10.18637/jss.v098.i16
[^3]: GitHub releases, JuliaStats/Distributions.jl (v0.20.0 2019-05-17, v0.24.0 2020-10-07, v0.25.0 2021-05-02, v0.25.129 2026-06-26). https://github.com/JuliaStats/Distributions.jl/releases
[^4]: Rmath-julia — R's standalone math library packaged for Julia. https://github.com/JuliaStats/Rmath-julia
[^5]: PDMats.jl — positive-definite matrix types. https://github.com/JuliaStats/PDMats.jl
[^6]: DistributionsAD.jl — automatic-differentiation compatibility layer maintained by TuringLang. https://github.com/TuringLang/DistributionsAD.jl

## Tags

julia, statistics, probability-distributions, data-science, random-sampling, maximum-likelihood, scientific-computing, numerical-computing, library
