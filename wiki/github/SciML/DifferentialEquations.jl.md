# SciML/DifferentialEquations.jl

> The umbrella package for the SciML differential-equation solver ecosystem — one `solve(prob, alg)` interface over ODEs, SDEs, DDEs, DAEs, and jump processes.

[GitHub repo](https://github.com/SciML/DifferentialEquations.jl) ·
[Official docs](https://docs.sciml.ai/DiffEqDocs/stable/) ·
[License: MIT](https://github.com/SciML/DifferentialEquations.jl/blob/master/LICENSE.md)

## Overview

DifferentialEquations.jl, started by Chris Rackauckas in 2016[^1], is the front door to the SciML organization's solver stack. It is a **metapackage**: it contains almost no solver code itself, instead re-exporting OrdinaryDiffEq.jl, StochasticDiffEq.jl, DelayDiffEq.jl, Sundials.jl, JumpProcesses.jl, and others behind a single problem/solve interface. The suite covers an unusually wide equation zoo — ODEs, SDEs, DAEs, DDEs, random ODEs, jump diffusions, and hybrid discrete/continuous systems — where most competing libraries cover one or two.

The claim to fame is performance with modern methods: native Julia implementations (e.g. `Tsit5`, `Vern9`, `Rodas5P`) frequently outperform the classic Hairer Fortran and Sundials C codes that other ecosystems wrap, while those classic codes remain available behind the same interface for one-line benchmarking[^2]. The suite also anchors the "scientific machine learning" stack: adjoint sensitivity analysis (SciMLSensitivity.jl) and neural differential equations (DiffEqFlux.jl) are built on its solvers.

The defining tension is breadth versus weight. The metapackage pulls a very large dependency tree, and Julia's compile-latency model means the cost is paid at load and first solve. Most of the ecosystem's engineering effort since ~2022 has gone into fighting this (sub-package splits, precompilation), and the standard production advice is to *not* depend on DifferentialEquations.jl directly — see Production Notes.

## Getting Started

```julia
using Pkg
Pkg.add("DifferentialEquations")
```

```julia
using DifferentialEquations

function lorenz!(du, u, p, t)
    du[1] = 10.0 * (u[2] - u[1])
    du[2] = u[1] * (28.0 - u[3]) - u[2]
    du[3] = u[1] * u[2] - (8 / 3) * u[3]
end

u0 = [1.0, 0.0, 0.0]
prob = ODEProblem(lorenz!, u0, (0.0, 100.0))
sol = solve(prob, Tsit5())   # or solve(prob) for automatic stiffness handling

sol(50.0)      # dense-output interpolation at t = 50, no re-solve
sol.u[end]     # final state
```

Python and R bindings exist as `diffeqpy` and `diffeqr`, driving the same Julia solvers over a bridge — usable, but you inherit a Julia installation and its startup cost.

## Architecture / How It Works

The interface layer is **SciMLBase.jl**: it defines problem types (`ODEProblem`, `SDEProblem`, `DDEProblem`, `DAEProblem`, `BVProblem`, `JumpProblem`, ...), the `solve` function, and the solution interface. Solver packages implement `solve` dispatches for their algorithm types. This is classic Julia multiple-dispatch design: `solve(prob, Tsit5())` and `solve(prob, CVODE_BDF())` hit entirely different packages (pure-Julia OrdinaryDiffEq vs. the Sundials C wrapper) through one call signature, which is what makes the "switch methods by changing one line" story real.

Key mechanics:

- **In-place vs. out-of-place**: `f!(du, u, p, t)` mutates a preallocated buffer (the fast path for medium/large systems); `f(u, p, t)` returning an `SVector` from StaticArrays is faster for small (< ~20-dim) systems.
- **Default algorithm selection**: `solve(prob)` without an algorithm uses a polyalgorithm that starts explicit and switches to a stiff (Rosenbrock/BDF) method on detected stiffness.
- **Jacobians**: computed via ForwardDiff automatic differentiation by default; sparsity can be detected symbolically via Symbolics.jl and exploited with colored differencing (SparseDiffTools.jl). Linear solves inside implicit methods are pluggable through LinearSolve.jl.
- **Events**: `ContinuousCallback` (root-finding on a condition, e.g. bouncing-ball contact) and `DiscreteCallback` handle hybrid dynamics; this callback system is more complete than most ecosystems' event handling.
- **Dense output**: solutions carry method-specific interpolants, so `sol(t)` is a continuous function, not a lookup into saved points.
- **Ensembles/GPU**: `EnsembleProblem` parallelizes over threads, processes, or GPUs (DiffEqGPU.jl) for parameter sweeps and Monte Carlo SDE runs.

Since 2024, OrdinaryDiffEq.jl itself has been split into per-solver-family sub-libraries (`OrdinaryDiffEqTsit5`, `OrdinaryDiffEqRosenbrock`, etc.) so downstream users can load only the methods they need[^3].

## Production Notes

**Don't depend on the metapackage.** DifferentialEquations.jl exists for interactive use and tutorials. In a package or deployed system, depend on the specific solver library (`OrdinaryDiffEq`, or narrower still `OrdinaryDiffEqTsit5`) — the difference is minutes vs. seconds of install/precompile time and a far smaller dependency surface. This is the single most repeated piece of advice in the community, and the docs say it too.

**Latency (TTFX).** Julia pays compilation cost at first call. Julia 1.9+ native precompilation caching improved first-solve time dramatically, but a cold `using DifferentialEquations` is still heavy, and any custom `f` triggers fresh compilation. For CLI-style batch jobs that start a fresh process per run, this overhead can dominate; long-running processes amortize it to zero.

**AD type footgun.** Implicit solvers push ForwardDiff `Dual` numbers through your `f`. If you preallocate caches as `Vector{Float64}` or otherwise hardcode `Float64` inside `f`, you get a confusing `MethodError`/conversion error. Write type-generic code or use `PreallocationTools.jl` for dual-compatible caches.

**Tolerances.** Defaults (`reltol = 1e-3`, `abstol = 1e-6`) are chosen for speed and are looser than MATLAB's users often assume; for publication-grade or chaotic systems, tighten them explicitly and check convergence.

**Solver choice matters.** For large stiff systems, `Rodas5P` degrades past a few hundred equations; `FBDF`, `TRBDF2`, or Sundials `CVODE_BDF` with a Krylov linear solver scale better. `dt <= dtmin` warnings almost always mean a stiff problem given to a non-stiff solver, a discontinuity the integrator can't step over (add `tstops` or a callback), or an unstable model — not a solver bug.

**Version churn.** The ecosystem moves fast and the package count is high; incompatible version resolution across SciMLBase / OrdinaryDiffEq / SciMLSensitivity does happen after major releases. Check in your `Manifest.toml` for reproducibility. The v8 release carried many breaking changes; the migration notes live in OrdinaryDiffEq's NEWS.md[^3].

**PDEs are not first-class.** Despite "(S)PDEs" in the description, this is a method-of-lines timestepper: you discretize space yourself (or via ModelingToolkit/MethodOfLines.jl, which has real limitations at scale) and hand the resulting ODE/DAE system over. It is not a competitor to dedicated FEM frameworks.

## When to Use / When Not

**Use when:**
- You need stiff solvers, event handling, DAEs, delays, or SDEs beyond what `scipy.integrate` offers.
- You solve the same system many times (parameter estimation, ensembles, MCMC) — Julia's compile-once-run-fast model pays off exactly here.
- You need gradients through the solver (adjoint sensitivity, neural ODEs) in the SciML stack.
- You want to benchmark modern vs. classic (Sundials, radau) methods behind one interface.

**Avoid when:**
- Your team is not otherwise using Julia and the problem is a simple non-stiff ODE — `scipy` or `diffrax` keeps you in one language.
- You need fast cold-start, short-lived processes (serverless, per-run CLI) — startup latency dominates.
- Your real problem is a large-scale PDE — use a proper FEM/FVM framework and only borrow a timestepper if needed.
- You need hard-real-time or embedded deployment; Julia's runtime and GC are the wrong shape for that.

## Alternatives

- scipy/scipy — `solve_ivp` covers casual non-stiff/stiff ODEs in Python with zero extra setup; use it when solver performance is not the bottleneck.
- patrick-kidger/diffrax — JAX-native ODE/SDE solvers with autodiff built in; use it when your model already lives in JAX.
- rtqichen/torchdiffeq — PyTorch neural-ODE solvers; use for neural ODEs inside a Torch training loop, not for general stiff scientific work.
- LLNL/sundials — the battle-tested C library (CVODE/IDA/ARKODE); use it directly when embedding into a C/C++/Fortran production code.
- boostorg/odeint — header-only C++ ODE integration; use for lightweight in-process C++ integration without external dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016 | Initial development under JuliaDiffEq (repo created 2016-05)[^4]. |
| — | 2017-05 | JORS paper: "DifferentialEquations.jl — A Performant and Feature-Rich Ecosystem"[^1]. |
| — | 2020 | JuliaDiffEq org rebrands as SciML, widening scope to scientific ML[^5]. |
| 7.0 | 2022-02 | Major release; ecosystem consolidation on SciMLBase interfaces. |
| — | 2024 | OrdinaryDiffEq split into per-family sub-libraries to cut load times[^3]. |
| 8.0 | 2026 | Many breaking changes; migration documented in OrdinaryDiffEq NEWS.md[^3]. |

## References

[^1]: Rackauckas & Nie, "DifferentialEquations.jl — A Performant and Feature-Rich Ecosystem for Solving Differential Equations in Julia", Journal of Open Research Software, 2017. https://openresearchsoftware.metajnl.com/articles/10.5334/jors.151
[^2]: Rackauckas, "A Comparison Between Differential Equation Solver Suites In MATLAB, R, Julia, Python, C, Mathematica, Maple, and Fortran" — 2018. http://www.stochasticlifestyle.com/comparison-differential-equation-solver-suites-matlab-r-julia-python-c-fortran/
[^3]: OrdinaryDiffEq.jl NEWS.md — sub-library split and v8 migration notes. https://github.com/SciML/OrdinaryDiffEq.jl/blob/master/NEWS.md
[^4]: DifferentialEquations.jl repository (created 2016-05-11). https://github.com/SciML/DifferentialEquations.jl
[^5]: SciML — Open Source Software for Scientific Machine Learning. https://sciml.ai/

## Tags

julia, differential-equations, ode-solver, sde, dae, scientific-computing, scientific-machine-learning, numerical-methods, simulation, sciml, solver-suite
