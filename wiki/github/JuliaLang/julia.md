# JuliaLang/julia

> A JIT-compiled dynamic language for technical computing that tries to solve the "two-language problem" with multiple dispatch.

[GitHub repo](https://github.com/JuliaLang/julia) ·
[Official website](https://julialang.org/) ·
[License: MIT](https://github.com/JuliaLang/julia/blob/master/LICENSE.md)

## Overview

Julia is a dynamically typed language that compiles to native code through LLVM, aimed at numerical and scientific computing. It was started at MIT by Jeff Bezanson, Stefan Karpinski, Viral B. Shah, and Alan Edelman, first announced publicly in 2012, with the stable 1.0 released in August 2018[^1]. Its pitch is the "two-language problem": prototype in a high-level language (Python, R, MATLAB) and rewrite hot loops in C/Fortran. Julia argues you can stay in one language and still get C-comparable speed, because the same code you write interactively is JIT-compiled and specialized on the concrete types it is called with.

The defining feature is **multiple dispatch**: functions are generic, and which method runs is chosen at call time from the runtime types of *all* arguments, not just a receiver object. This is not syntactic sugar — it is the organizing principle of the whole ecosystem. Numeric types, units, dual numbers for autodiff, GPU arrays, and measurement-with-uncertainty types compose across unrelated packages precisely because they slot into a shared dispatch hierarchy rather than into class inheritance trees. Packages that were never written to know about each other frequently work together unchanged.

The central tension is **latency versus performance**. Because Julia specializes and compiles on first call, the first invocation of a plotting or dataframe workflow historically took many seconds — the notorious "time to first plot" (TTFP). Steady-state performance is excellent; the cost is front-loaded and felt most by interactive users and short-lived scripts. Much of Julia's engineering since 2020 has gone into attacking this, and it is materially better than it was, but "startup and first-call latency" remains the honest weak spot relative to interpreted languages.

## Getting Started

The recommended installer is `juliaup`, a version manager that installs and updates the `julia` binary[^2]:

```bash
# macOS / Linux
curl -fsSL https://install.julialang.org | sh
# Windows: winget install julia -s msstore
```

```julia
# multiple dispatch: one generic function, methods chosen on argument types
struct Point{T}
    x::T
    y::T
end

dist(a::Point, b::Point) = hypot(a.x - b.x, a.y - b.y)

julia> dist(Point(0.0, 0.0), Point(3.0, 4.0))
5.0
```

Package management is built in. Press `]` at the REPL to enter Pkg mode:

```julia
julia> ]                       # enter the package REPL
(@v1.11) pkg> add DataFrames   # resolves + installs into the active environment
```

## Architecture / How It Works

The compilation pipeline: source is parsed (as of 1.10 by the pure-Julia `JuliaSyntax` parser), lowered to an SSA-form intermediate representation, run through **type inference**, then lowered to LLVM IR and JIT-compiled to native code[^3]. Inference is the load-bearing pass: if the compiler can prove concrete types flow through a function, it emits tight monomorphized machine code; if it cannot, values are boxed and dispatched dynamically at runtime, which is where most Julia performance problems come from.

- **Type system.** Types are first-class runtime values. There is a nominal abstract/concrete hierarchy (`Integer` <: `Real` <: `Number`), parametric types (`Array{Float64,2}`), and union types. Only concrete leaf types have instances. Method specialization keys off concrete types, so `sum(::Vector{Int})` and `sum(::Vector{Float64})` compile to separate native functions.
- **Multiple dispatch runtime.** Every generic function owns a method table; a call site resolves to the most specific applicable method. When resolvable at compile time this is devirtualized to a direct call; otherwise it becomes a runtime table lookup.
- **Memory model.** A non-moving, mark-sweep, generational garbage collector. Arrays are column-major and 1-indexed. There is no compaction, so long-lived programs can fragment.
- **Parallelism.** Four distinct models ship in the standard library: coroutine-style `Task`s, native `Threads` (composable task-based multithreading, matured through the 1.x line), `Distributed` multiprocessing, and a GPU story via the external CUDA.jl / AMDGPU.jl / Metal.jl stack built on the same array abstractions.

The ecosystem's crown jewels — DifferentialEquations.jl / the SciML stack, JuMP for optimization, Flux and Lux for ML, Turing for probabilistic programming — lean entirely on dispatch-based composability rather than on framework-style plugin APIs.

## Production Notes

**Compile latency is the recurring operational cost.** Julia 1.9 (2023) added native-code caching to disk ("package images"), so precompiled packages now ship cached machine code, not just parsed IR — this is the single biggest latency improvement in the language's history[^4]. Combined with parallel precompilation in 1.10, TTFP dropped from tens of seconds to a few. It is not zero: interactive workflows and CI still pay a precompile tax after every package update, and short CLI scripts remain a poor fit.

**Type instability is the #1 performance footgun.** Code that is correct but lets the compiler lose track of a concrete type silently falls back to boxed dynamic dispatch and can run orders of magnitude slower. The tools to find this are `@code_warntype`, `@inferred`, and `JET.jl` / `Cthulhu.jl`. Common culprits: non-`const` global variables, containers of abstract element type (`Vector{Any}`), and functions whose return type depends on a value rather than a type.

**Invalidations.** Loading a package that adds methods to functions other packages already specialized can invalidate previously compiled code, forcing recompilation and inflating load times. `SnoopCompile.jl` diagnoses this; it mostly bites package authors, not end users.

**Static deployment is weak but improving.** Shipping a small standalone binary is historically hard — `PackageCompiler.jl` builds sysimages and app bundles, but they are large (the runtime and LLVM come along). A first-class static compiler (`juliac`) is under active development for the 1.12 line; treat sub-megabyte native binaries as not-yet-solved.

**Ecosystem maturity is uneven.** Core numerical/scientific packages are excellent and battle-tested. Outside that core, package count is far smaller than Python's, and unmaintained or one-maintainer packages are common. There is no equivalent of the pandas/NumPy install base for general-purpose glue work.

**Versioning.** 1.6 was the long-term-support (LTS) release for years; 1.10 became the new LTS. Point releases are largely non-breaking under semver, but performance characteristics and default behaviors do shift between minor versions, so pin the Julia version in CI.

## When to Use / When Not

**Use when:**
- You do heavy numerical, scientific, or optimization work and are hitting the two-language wall in Python/R/MATLAB.
- You need C-class inner-loop performance but want to keep writing high-level, interactive code.
- You benefit from ecosystem composability — autodiff, uncertainty, units, or GPU arrays flowing through third-party solvers unchanged.
- Long-running compute processes (simulations, HPC jobs) where startup latency amortizes to nothing.

**Avoid when:**
- You need fast-starting short-lived scripts or CLI tools — startup + first-call latency dominates.
- You need a small self-contained distributable binary today.
- You need the breadth of a large, mature general-purpose package ecosystem for non-numeric glue code.
- Your team's problem is web/systems/orchestration rather than math-heavy compute.

## Alternatives

- python/cpython — vastly larger ecosystem and mindshare; NumPy/SciPy/PyTorch cover most numeric needs. Use it when ecosystem breadth and hiring matter more than staying in one language for speed.
- rust-lang/rust — memory-safe, no GC, excellent for systems and performance-critical libraries. Use it when you need deterministic performance and static binaries rather than interactive numeric exploration.
- JuliaLang-adjacent MATLAB (proprietary) — Julia's direct inspiration and competitor. Use it when you are locked into Simulink/toolboxes or an existing MATLAB codebase.
- google/jax (Python) — composable autodiff + XLA JIT for array programs. Use it when your workload is ML/array-transform-centric and you want the Python ecosystem around it.
- R (r-source) — unmatched for statistics and data analysis packages. Use it when statistical modeling breadth outweighs raw speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2012-02 | First public announcement / release[^1]. |
| 0.6 | 2017-06 | Type-system overhaul; last major pre-1.0 series. |
| 1.0 | 2018-08 | First stable release; language semantics frozen under semver[^1]. |
| 1.6 | 2021-03 | Long-term-support release; parallel precompilation, faster loading. |
| 1.9 | 2023-05 | Native code caching to disk (package images) — major latency cut[^4]. |
| 1.10 | 2023-12 | New `JuliaSyntax` parser, parallel precompile; new LTS. |
| 1.11 | 2024-10 | `Memory` type, further compiler and GC work. |

## References

[^1]: Bezanson, Karpinski, Shah, Edelman, "Julia: A Fresh Approach to Numerical Computing," SIAM Review 2017; language announced 2012, 1.0 released 2018-08. https://julialang.org/blog/2012/02/why-we-created-julia/ and https://julialang.org/blog/2018/08/one-point-zero/
[^2]: `juliaup` installer and version manager. https://github.com/JuliaLang/juliaup
[^3]: Julia developer documentation — compiler and internals. https://docs.julialang.org/en/v1/devdocs/eval/
[^4]: "Julia 1.9 Highlights" — native code caching / package images. https://julialang.org/blog/2023/04/julia-1.9-highlights/

## Tags

julia, programming-language, jit-compiler, scientific-computing, numerical-computing, multiple-dispatch, llvm, hpc, machine-learning, technical-computing
