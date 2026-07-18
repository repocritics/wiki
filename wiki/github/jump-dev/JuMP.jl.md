# jump-dev/JuMP.jl

> An algebraic modeling language for mathematical optimization embedded in Julia — write the math once, solve it with any of ~50 solvers.

[GitHub repo](https://github.com/jump-dev/JuMP.jl) ·
[Official website](https://jump.dev) ·
[License: MPL-2.0](https://github.com/jump-dev/JuMP.jl/blob/master/LICENSE.md)

## Overview

JuMP is a domain-specific modeling language for mathematical optimization — linear, mixed-integer, conic, semidefinite, and nonlinear programs — expressed as ordinary Julia code via macros. It began in 2012 at MIT's Operations Research Center (Miles Lubin, Iain Dunning, later Joey Huchette) as an open-source answer to commercial modeling languages like AMPL and GAMS[^1]. The pitch: AMPL-like syntax without the license fee, and without the wall between the model and a general-purpose programming language.

Its 2,454 stars understate its position. JuMP's audience is operations researchers, energy-system modelers, and quantitative engineers, not web developers; within that niche it is the dominant open-source modeling layer — a NumFOCUS-sponsored project with an annual JuMP-dev workshop, its own software-journal papers[^1][^2], and a commercially significant ecosystem (Gurobi, CPLEX, and Mosek all have maintained Julia wrappers). Development is active: pushed to daily as of mid-2026, with a deliberately clean issue tracker (11 open issues — support is routed to the community forum instead).

The defining tension is speed-of-expression vs. the two-layer abstraction underneath. JuMP models look like textbook math, but every constraint passes through MathOptInterface (MOI), a solver-abstraction layer with its own type system, reformulation "bridges," and caching semantics[^3]. When things are fast and solvable, you never see MOI. When a model builds slowly or a solver rejects a constraint form, you end up debugging the abstraction.

## Getting Started

```julia
import Pkg; Pkg.add(["JuMP", "HiGHS"])   # HiGHS = open-source LP/MIP solver
```

```julia
using JuMP, HiGHS

model = Model(HiGHS.Optimizer)
@variable(model, x >= 0)
@variable(model, 0 <= y <= 3)
@objective(model, Min, 12x + 20y)
@constraint(model, c1, 6x + 8y >= 100)
@constraint(model, c2, 7x + 12y >= 120)
optimize!(model)

@assert is_solved_and_feasible(model)
value(x), value(y), objective_value(model)   # (15.0, 1.25, 205.0)
```

Swapping `HiGHS.Optimizer` for `Gurobi.Optimizer` or `Ipopt.Optimizer` is the entire migration story between solvers, provided the solver supports your problem class[^4].

## Architecture / How It Works

JuMP is a macro frontend over **MathOptInterface (MOI)**, a separate package that defines a standard form for optimization problems: variables, functions (affine, quadratic, nonlinear), and sets (nonnegatives, second-order cones, PSD cones, integers)[^3]. Every solver wrapper (HiGHS.jl, Ipopt.jl, Gurobi.jl, ~50 others) implements the MOI API, not a JuMP API.

- **Macros, not operator overloading.** `@constraint(model, sum(c[i]*x[i] for i in I) <= b)` is rewritten at parse time into efficient in-place expression construction. Writing the same sum without macros triggers O(n²) operator-overloading behavior — a classic performance cliff[^5].
- **Bridges.** If a solver does not natively support a constraint form (e.g., you write an interval constraint, the solver wants two inequalities), MOI automatically reformulates it through a shortest-path search over bridge transformations. Convenient, and occasionally the silent source of a much larger problem than the one you wrote.
- **CachingOptimizer vs. direct mode.** By default JuMP keeps a full copy of the model outside the solver so it can rebuild on demand; `direct_model()` skips the copy for memory-critical or rapid-remodification workflows, at the cost of restricting operations to what the solver natively supports.
- **Nonlinear.** Legacy `@NL*` macros used a separate expression tree and built-in reverse-mode AD. Since v1.15 (2023), nonlinear expressions flow through the standard `@constraint` / `@objective` macros as `ScalarNonlinearFunction`, unifying the two systems[^6]; the `@NL*` macros still work but are the legacy path.
- **Generic number types.** `GenericModel{T}` supports arbitrary-precision and rational arithmetic end-to-end with compatible solvers — rare among modeling languages.

## Production Notes

- **Check status before reading results.** `value()` after a failed solve throws or returns garbage. The idiom is `is_solved_and_feasible(model)` or explicit `termination_status` / `primal_status` checks; `OPTIMAL` (global) and `LOCALLY_SOLVED` (NLP) are different statuses, and treating them interchangeably is a common correctness bug.
- **First-solve latency.** Julia JIT-compiles on first execution, so the first `optimize!` in a fresh session carries seconds of compilation overhead. Julia 1.9+ native code caching reduced this substantially, but short-lived CLI-style batch jobs still pay it; long-running services and notebook workflows amortize it away.
- **Model build time is a real cost.** For models with millions of constraints, building can rival solving. Known mitigations: disable string names (`set_string_names_on_creation(model, false)`), use `add_to_expression!` instead of `+=` on expressions, and prefer the macro forms everywhere[^5].
- **The 0.18 → 0.19 scar.** The 2019 MOI rewrite broke nearly every existing model and solver wrapper; ecosystem migration took years, and old StackOverflow answers from that era still mislead. Post-1.0 (2022) the project commits to semver and has held to it — upgrade pain since has been minor[^2].
- **Solver licensing is separate from JuMP licensing.** JuMP is MPL-2.0, but the strongest MIP solvers (Gurobi, CPLEX) are commercial with their own license servers; HiGHS is the best fully open option and has closed much of the gap for mid-size problems. Budget for solver licenses, not for JuMP.
- **Dual sign conventions.** JuMP uses conic duality conventions; duals of `<=` constraints in minimization problems are nonpositive. Cross-checking against textbook LP duality without reading the manual is a recurring trap.

## When to Use / When Not

**Use when:**
- You solve LP/MIP/conic/NLP problems and want solver portability — prototype on HiGHS, deploy on Gurobi, without rewriting the model.
- The model interleaves with real computation (simulation, ML, data prep) and a standalone modeling language would force awkward data plumbing.
- You need problem classes commercial AMLs handle poorly (SDP, complementarity, arbitrary-precision arithmetic).

**Avoid when:**
- Your team is Python-only and unwilling to adopt Julia; Pyomo and cvxpy are weaker modeling layers but zero-friction organizationally.
- You run many short-lived cold-start jobs where JIT latency dominates total runtime.
- Your problem is pure convex programming with a DSL fit — cvxpy's DCP verification catches modeling errors JuMP will happily pass to the solver.
- You need optimal-control tooling (collocation, integrators) out of the box — CasADi is purpose-built for that.

## Alternatives

- Pyomo/pyomo — use instead when the organization is Python-committed; similar breadth, slower model construction, larger install base.
- cvxpy/cvxpy — use instead for convex-only problems where DCP-verified correctness matters more than problem-class breadth.
- coin-or/pulp — use instead for simple LP/MIP in Python with minimal dependencies and no nonlinear or conic needs.
- casadi/casadi — use instead for optimal control and NLP with heavy algorithmic-differentiation needs.
- jump-dev/Convex.jl — use instead within Julia when you want DCP-style convexity verification on top of the same MOI solver ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013 | Initial release from MIT ORC; LP/MIP via MathProgBase[^1]. |
| 0.19 | 2019-02 | MathOptInterface replaces MathProgBase; largest breaking rewrite in project history[^3]. |
| 1.0 | 2022-03 | Stability commitment after a decade of 0.x; semver adherence since[^7]. |
| 1.15 | 2023-08 | Unified nonlinear interface — nonlinear expressions in standard macros[^6]. |
| 1.x | 2026 | Steady minor releases; active daily development on master. |

## References

[^1]: Dunning, Huchette, Lubin, "JuMP: A Modeling Language for Mathematical Optimization" — SIAM Review 59(2), 2017. https://dx.doi.org/10.1137/15M1020575
[^2]: Lubin, Dowson, Dias Garcia, Huchette, Legat, Vielma, "JuMP 1.0: Recent improvements to a modeling language for mathematical optimization" — Mathematical Programming Computation 15, 2023. https://www.doi.org/10.1007/s12532-023-00239-3
[^3]: MathOptInterface documentation. https://jump.dev/MathOptInterface.jl/stable/
[^4]: JuMP docs, "Installation Guide — Supported solvers". https://jump.dev/JuMP.jl/stable/installation/
[^5]: JuMP docs, "Performance tips". https://jump.dev/JuMP.jl/stable/tutorials/getting_started/performance_tips/
[^6]: JuMP docs, "Nonlinear Modeling". https://jump.dev/JuMP.jl/stable/manual/nonlinear/
[^7]: JuMP v1.0.0 release. https://github.com/jump-dev/JuMP.jl/releases/tag/v1.0.0

## Tags

julia, mathematical-optimization, linear-programming, mixed-integer-programming, nonlinear-programming, conic-programming, modeling-language, operations-research, solver-agnostic, dsl
