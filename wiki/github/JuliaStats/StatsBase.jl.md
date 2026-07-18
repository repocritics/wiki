# JuliaStats/StatsBase.jl

> The descriptive-statistics workhorse of the Julia ecosystem — weighted
> statistics, sampling, ranking, and histograms that nearly every Julia data
> package depends on, still versioned 0.x after a decade.

[GitHub repo](https://github.com/JuliaStats/StatsBase.jl) ·
[Documentation](https://juliastats.org/StatsBase.jl/stable/) ·
[License: MIT](https://github.com/JuliaStats/StatsBase.jl/blob/master/LICENSE.md)

## Overview

StatsBase.jl provides the layer of statistics that Julia's `Statistics`
standard library deliberately leaves out: weighted means/variances/quantiles,
higher-order moments (skewness, kurtosis), counting (`countmap`), ranking
(`tiedrank`, `competerank`), autocorrelation and partial autocorrelation,
sampling with and without replacement, empirical CDFs, histograms via
`fit(Histogram, ...)`, and data transforms like `zscore`. It was started by
Dahua Lin in February 2013[^1], making it one of the oldest continuously
maintained packages in the Julia ecosystem, and it sits inside the JuliaStats
organization alongside Distributions.jl, GLM.jl, and HypothesisTests.jl.

Its raw GitHub numbers (610 stars, 196 forks) badly understate its position.
Julia packages are consumed through the General registry, not GitHub stars, and
StatsBase is one of the most-depended-upon packages in that registry — a
transitive dependency of most of the data/science stack. That centrality is
also its defining tension: after 13 years and 70+ tagged releases the package
is still at v0.34.x[^2]. Under Julia's semver conventions every 0.x minor bump
is breaking, so a StatsBase minor release historically required a coordinated
compat-bound update across thousands of downstream packages. A 1.0 release and
a partial merge into the `Statistics` stdlib have been discussed for years
without shipping.

The second structural shift worth knowing: the abstract modeling API that
StatsBase originally defined (`coef`, `nobs`, `r2`, `loglikelihood`,
`predict`, `residuals`, the `StatisticalModel`/`RegressionModel` types) was
extracted into the dependency-free StatsAPI.jl[^3] so that lightweight packages
can implement the interface without pulling in StatsBase itself. StatsBase now
re-exports those functions, so old code keeps working, but new interface code
should target StatsAPI.

## Getting Started

```julia
using Pkg; Pkg.add("StatsBase")
```

```julia
using StatsBase

x = randn(1000)
w = Weights(rand(1000))

mean(x, w)                        # weighted mean
std(x, w; corrected=false)        # generic Weights define no bias correction
countmap([1, 2, 2, 3])            # Dict(1 => 1, 2 => 2, 3 => 1)
sample(1:100, 10; replace=false)  # sampling without replacement
h = fit(Histogram, x; nbins=20)   # histogram as a fitted object
F = ecdf(x); F(0.0)               # empirical CDF, callable
zscore(x)                         # standardize
corspearman(x, randn(1000))       # rank correlation
```

## Architecture / How It Works

StatsBase is a pure-Julia library with a deliberately thin dependency stack
(StatsAPI, DataStructures, SortingAlgorithms, and stdlibs). Its main design
ideas:

- **Weights as types, not keyword arguments.** `AbstractWeights` has concrete
  subtypes — `Weights`, `AnalyticWeights`, `FrequencyWeights`,
  `ProbabilityWeights`, `UnitWeights` — and the *type* determines the
  statistical semantics. `var`/`std` apply a different bias-correction formula
  per weight type, and plain `Weights` intentionally has no corrected
  estimator (calling `var(x, w; corrected=true)` on it throws)[^4]. This is
  principled but routinely surprises users coming from R/pandas, where a
  weight is just a vector.
- **Extending stdlib generics.** Weighted `mean`, `var`, `std`, `quantile`,
  and `median` are implemented as new methods on the `Statistics` stdlib
  functions, so `using StatsBase` transparently upgrades functions you already
  call. The abstract model API comes from StatsAPI and is re-exported.
- **`fit` as the front door.** Histograms and data transforms
  (`ZScoreTransform`, `UnitRangeTransform`) follow the `fit(T, data...)` →
  object → `transform`/`reconstruct` pattern shared across JuliaStats, rather
  than one-shot functions.
- **Sampling algorithms selected by heuristic.** `sample` and `wsample`
  dispatch among several algorithms (alias tables, Fisher–Yates,
  reservoir-style sampling) depending on population size, sample size,
  replacement, and whether ordered output is requested[^5]. You get
  near-optimal algorithmic choices without naming them, at the cost of
  performance characteristics that shift with input sizes.

## Production Notes

- **The 0.x compat treadmill is the real operational cost.** Because minor
  bumps are breaking, one package in your dependency tree capping
  `StatsBase = "0.33"` can hold your entire environment back from 0.34 — and
  with it, transitively pin other packages to old versions. When `Pkg.resolve`
  produces mysteriously old manifests, a StatsBase compat cap is a common
  culprit. `Pkg.status --outdated` and `Pkg.why` are the diagnostic tools.
- **v0.33 → v0.34 (2023) removed long-deprecated functionality**; code that
  had been ignoring deprecation warnings for years broke on upgrade. The 0.34
  series itself (2023–2026) has been patch-only and conservative — 12 patches
  in three years[^2], which reads as "stable and maintained" rather than
  "stagnant", but also means long-requested features move slowly. 221 open
  issues reflect a wide user base against a small reviewer pool.
- **Weight-type semantics are a correctness footgun, not just an API quirk.**
  `FrequencyWeights` and `ProbabilityWeights` give different corrected
  variances on the same numbers. Picking `Weights` "because it's the obvious
  name" and then hitting the corrected-variance error — or silently passing
  `corrected=false` — is the most common way to get subtly wrong uncertainty
  estimates out of this package.
- **Histogram edge semantics.** `fit(Histogram, ...)` defaults to
  `closed=:left` bins; code ported from systems assuming right-closed bins
  (or from older StatsBase versions) can miscount boundary values. Pass
  `closed` explicitly when reproducing results from other tools.
- **Performance is good but not vectorized-C good.** Kernels are plain Julia
  loops: fast, type-stable, and AD-unfriendly in places (in-place accumulation).
  For datasets that don't fit memory or need single-pass computation, StatsBase
  has nothing — that niche belongs to OnlineStats.jl.

## When to Use / When Not

**Use when:**
- You're in Julia and need anything beyond stdlib `mean`/`var`/`cor` —
  weighted statistics, ranks, sampling, ECDFs, histograms.
- You need statistically distinct weight semantics (frequency vs probability
  vs analytic) handled correctly rather than by hand.
- You're implementing a statistical model type and want ecosystem-standard
  accessors (via the re-exported StatsAPI interface).

**Avoid when:**
- Stdlib `Statistics` already covers you — StatsBase is light, but every
  dependency is compat surface on the 0.x treadmill.
- You need streaming/out-of-core statistics — use OnlineStats.jl.
- You need distribution fitting, hypothesis tests, or regression — those live
  in Distributions.jl, HypothesisTests.jl, and GLM.jl; StatsBase is
  descriptive-statistics only.

## Alternatives

- JuliaLang/Statistics.jl — use instead when unweighted `mean`/`median`/
  `var`/`cor` is all you need; it's the stdlib with zero added compat surface.
- joshday/OnlineStats.jl — use instead for single-pass, mergeable, streaming
  statistics on data too large for memory.
- JuliaStats/Distributions.jl — use instead when you want parametric
  distributions and MLE fitting rather than empirical/descriptive statistics.
- JuliaStats/HypothesisTests.jl — use instead (or alongside) when the goal is
  significance testing, not summarization.
- scipy/scipy — use instead when you're in Python; `scipy.stats` covers most
  of the same descriptive ground.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-02 | Repo created by Dahua Lin[^1]. |
| v0.25.0 | 2018-08 | Julia 1.0-era release during the 0.7→1.0 transition. |
| v0.33.0 | 2020-03 | Start of the long-lived 0.33 series (21 patch releases through 2022). |
| v0.34.0 | 2023-05 | Deprecation removals; breaking minor that required ecosystem-wide compat bumps. |
| v0.34.12 | 2026-06 | Latest release; 0.34 series has been patch-only since 2023[^2]. |

## References

[^1]: GitHub repository, created 2013-02-11; license credits "Copyright (c) 2012-2016: Dahua Lin, Simon Byrne, Andreas Noack, Douglas Bates, John Myles White, Simon Kornblith, and other contributors." https://github.com/JuliaStats/StatsBase.jl
[^2]: StatsBase.jl releases (v0.34.0 2023-05-01 … v0.34.12 2026-06-17). https://github.com/JuliaStats/StatsBase.jl/releases
[^3]: StatsAPI.jl — dependency-free statistics API definitions re-exported by StatsBase. https://github.com/JuliaStats/StatsAPI.jl
[^4]: StatsBase.jl docs, "Weight Vectors" — per-type bias correction semantics. https://juliastats.org/StatsBase.jl/stable/weights/
[^5]: StatsBase.jl docs, "Sampling from Population". https://juliastats.org/StatsBase.jl/stable/sampling/

## Tags

julia, statistics, descriptive-statistics, weighted-statistics, sampling, histograms, ranking, data-science, numerical-computing, library
