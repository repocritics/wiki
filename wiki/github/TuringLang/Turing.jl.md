# TuringLang/Turing.jl

> General-purpose probabilistic programming in Julia — Bayesian models as ordinary Julia functions, sampled with MCMC or fit variationally.

[GitHub repo](https://github.com/TuringLang/Turing.jl) ·
[Official website](https://turinglang.org) ·
[License: MIT](https://github.com/TuringLang/Turing.jl/blob/main/LICENCE)

## Overview

Turing.jl is a probabilistic programming language (PPL) embedded in Julia,
started at the University of Cambridge by Hong Ge in 2016 and formally
described at AISTATS 2018[^1]. A model is a Julia function annotated with the
`@model` macro; `~` statements declare random variables, and any Julia code —
loops, conditionals, calls into ODE solvers or neural network layers — can
appear in the model body. This "dynamic" design is the core bet: where Stan
restricts you to a fixed-shape model in a dedicated language compiled to C++,
Turing lets the model be arbitrary Julia, including models whose
dimensionality changes stochastically at runtime[^2].

The tradeoff is that flexibility pushes work onto the user and the compiler:
performance depends heavily on how the model is written, gradient-based
samplers depend on Julia's still-uneven automatic differentiation ecosystem,
and first-run latency inherits Julia's compile-time costs. Turing itself is a
facade — the front door to a dozen smaller, independently versioned TuringLang
packages[^3]. It is maintained primarily by academic researchers at
grant-funded institutions with explicitly limited triage capacity[^4], yet is
genuinely active (pushed within days as of mid-2026, ~2.2k stars, fortnightly
ecosystem newsletter, 2025 journal paper in ACM TOPML[^5]).

## Getting Started

Requires Julia (≥ 1.10.8 for current releases)[^4]:

```julia
julia> using Pkg; Pkg.add("Turing")
```

```julia
using Turing

@model function linear_regression(x)
    # Priors
    α ~ Normal(0, 1)
    β ~ Normal(0, 1)
    σ² ~ truncated(Cauchy(0, 3); lower=0)

    # Likelihood
    μ = α .+ β .* x
    y ~ MvNormal(μ, σ² * I)
end

x, y = rand(10), rand(10)
posterior = linear_regression(x) | (; y = y)   # condition on observed y
chain = sample(posterior, NUTS(), 1000)
```

The `|` conditioning syntax means one model definition serves both generative
simulation and inference.

## Architecture / How It Works

Turing.jl itself is mostly glue. The real machinery lives in sibling packages
under the TuringLang organization[^3]:

- **DynamicPPL.jl** — the model DSL. `@model` rewrites each `~` statement into
  a call that either *assumes* (samples a latent variable) or *observes*
  (accumulates log-likelihood), depending on what the model has been
  conditioned on; values and metadata live in a `VarInfo` structure threaded
  through model execution.
- **AbstractMCMC.jl** — the minimal sampler interface all samplers implement,
  including chain parallelism via `MCMCThreads()` / `MCMCDistributed()`.
- **AdvancedHMC.jl** — HMC/NUTS; **AdvancedMH.jl** — Metropolis–Hastings;
  **AdvancedPS.jl** — particle methods (SMC, particle Gibbs), which are what
  make stochastic control flow and varying dimension tractable.
- **Bijectors.jl** — maps constrained parameters (e.g. a positive `σ²`) to
  unconstrained space so HMC can operate freely; **AdvancedVI.jl** —
  variational inference; **Optimization.jl** (SciML) backs MLE/MAP;
  **MCMCChains.jl** — the `Chains` result type with R-hat/ESS diagnostics.

Gradient-based samplers need AD over the model's log density; the preferred
backends are ForwardDiff.jl and Mooncake.jl, with others available through
DifferentiationInterface.jl[^4]. Samplers compose: `Gibbs` can assign NUTS to
continuous parameter blocks and particle Gibbs to discrete ones — something
Stan cannot do at all without manual marginalization of discrete parameters.

## Production Notes

**Model style dominates performance.** A naive port of a BUGS/Stan model —
scalar observation loops, untyped containers, non-const globals — can be an
order of magnitude slower than an idiomatic one using vectorized multivariate
likelihoods and type-stable code. Profile the log density before blaming the
sampler.

**Choose the AD backend deliberately.** ForwardDiff (the default) scales with
parameter count; reverse-mode backends are the usual fix beyond roughly a
hundred parameters. Historically Zygote was fragile around mutation inside
models; Mooncake is the current preferred reverse-mode backend[^4]. Backend
switches can change both speed and whether a model differentiates at all.

**Compile latency.** The first `sample` call pays Julia's compilation cost —
often tens of seconds for a nontrivial model. Julia 1.9+ native caching helped
package loads, but per-model compilation remains, which makes Turing awkward
for short-lived scripts and CLI-style batch jobs.

**Interface churn.** Turing has lived on 0.x versioning for most of its life,
with breaking minor releases; sampler options, `VarInfo` internals, and AD
selection have all shifted. Pin versions and read `HISTORY.md` first[^6].

**Fragmented issue surface, limited capacity.** Bugs frequently belong to
DynamicPPL, AdvancedHMC, Bijectors, or an AD package rather than Turing.jl —
stack traces span four or five packages. Maintainers migrate misfiled issues,
but ask that new features be proposed before implementation; plan for slower
review turnaround than commercially backed projects[^4].

## When to Use / When Not

**Use when:**
- Your model composes with the Julia ecosystem — ODEs (SciML), neural network
  components, custom likelihoods written as plain Julia functions.
- You need discrete parameters or stochastic control flow sampled directly
  rather than marginalized by hand.
- You are already in Julia and want priors-to-diagnostics in one language, with
  generative and inference use from a single model definition.

**Avoid when:**
- You need a battle-hardened, heavily diagnosed HMC stack with maximal
  per-iteration speed on a static model — Stan remains the reference.
- Your team is Python-based; PyMC or NumPyro deliver comparable capability
  without a language switch.
- Startup latency matters (short scripts, serverless, frequent cold restarts).
- You need long-term interface stability an 0.x academic project cannot promise.

## Alternatives

- stan-dev/stan — use instead for the most mature, best-diagnosed HMC/NUTS
  stack when your model fits Stan's static language.
- pymc-devs/pymc — use instead when your team lives in Python and wants the
  largest Bayesian modeling community.
- pyro-ppl/numpyro — use instead for JAX-speed, GPU-friendly NUTS with a more
  restricted functional modeling style.
- pyro-ppl/pyro — use instead for deep probabilistic models and SVI at scale
  on PyTorch.
- blackjax-devs/blackjax — use instead for raw JAX sampler kernels when you
  will write the log density yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-04 | Repository created; project started at Cambridge (Hong Ge)[^7]. |
| — | 2018-04 | AISTATS 2018 paper, "Turing: A Language for Flexible Probabilistic Inference"[^1]. |
| — | 2020 | Model DSL and compiler internals factored out into DynamicPPL.jl[^2]. |
| — | 2025-02 | Journal paper accepted at ACM Transactions on Probabilistic Machine Learning[^5]. |
| — | 2026-07 | Actively maintained; ~2.2k stars, MIT license, main + `breaking` branch release flow[^7]. |

## References

[^1]: Hong Ge, Kai Xu, Zoubin Ghahramani, "Turing: A Language for Flexible Probabilistic Inference" — AISTATS 2018, PMLR 84:1682-1690. https://proceedings.mlr.press/v84/ge18b.html
[^2]: DynamicPPL.jl — the probabilistic-program DSL underlying Turing.jl. https://github.com/TuringLang/DynamicPPL.jl
[^3]: TuringLang documentation, "Where does Turing.jl sit in the TuringLang ecosystem?". https://turinglang.org
[^4]: Turing.jl README (maintenance capacity, AD backend policy, Julia version requirement). https://github.com/TuringLang/Turing.jl#readme
[^5]: Fjelde et al., "Turing.jl: a general-purpose probabilistic programming language" — ACM Trans. Probab. Mach. Learn., 2025. https://doi.org/10.1145/3711897
[^6]: Turing.jl changelog. https://github.com/TuringLang/Turing.jl/blob/main/HISTORY.md
[^7]: GitHub repository metadata (created 2016-04-29; pushed 2026-07-17). https://github.com/TuringLang/Turing.jl

## Tags

julia, bayesian-inference, probabilistic-programming, mcmc, hamiltonian-monte-carlo, variational-inference, statistics, machine-learning, scientific-computing
