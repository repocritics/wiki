# JuliaLang/PackageCompiler.jl

> Ahead-of-time compilation for Julia: bake packages into custom sysimages,
> standalone apps, and C-callable libraries — trading Julia's dynamism for
> startup speed and distributability.

[GitHub repo](https://github.com/JuliaLang/PackageCompiler.jl) ·
[Official docs](https://julialang.github.io/PackageCompiler.jl/dev/) ·
License: MIT

## Overview

PackageCompiler.jl is the official JuliaLang-org tool for ahead-of-time
compiling Julia code into three artifact shapes: **custom sysimages** (a
shared library the `julia` binary loads at startup, with your packages and
their compiled method specializations baked in), **apps** (a directory bundle
with an executable that runs on machines without Julia installed), and
**C libraries** (a shared library exposing `@ccallable` Julia functions to
C/C++/Fortran callers)[^1]. The current codebase is a ground-up rewrite by
Kristoffer Carlsson (initially published as PackageCompilerX) that replaced
Simon Danisch's original 2017 implementation and shipped as 1.0 in early
2020[^2].

The package exists because of Julia's defining tradeoff: a JIT-compiled
dynamic language whose "time to first X" latency (historically, time to first
plot) could run tens of seconds as methods compiled on first call. A sysimage
moves that compilation to build time. The catch is that everything baked into
a sysimage is frozen: package versions in the image shadow whatever the
active environment says, silently. This makes sysimages excellent for
deployment and CI, and a recurring source of confusion in interactive
development.

Its role has narrowed over time. Julia 1.9 (2023) introduced per-package
native code caching ("pkgimages"), which — with PrecompileTools workloads —
removed most of the latency motivation for custom sysimages[^3]. What
pkgimages did not replace is distribution: apps and C-library bundles for
Julia-less machines remain PackageCompiler's unique capability, until the
experimental `juliac` driver (Julia 1.12+) matures[^4]. At ~1.5k stars it
looks mid-sized, but stars understate it: this is org-owned infrastructure
most Julia deployment stories route through. Activity is maintenance-mode:
steady low-volume commits (last push 2026-06), 166 open issues.

## Getting Started

```
pkg> add PackageCompiler
```

A C toolchain is required for linking (gcc/clang on Linux/macOS; on Windows
PackageCompiler fetches a MinGW toolchain itself).

```julia
using PackageCompiler

# 1. Custom sysimage: bake Plots + a warm-up workload into a shared library
create_sysimage(
    ["Plots"];
    sysimage_path = "plots_sys.so",
    precompile_execution_file = "warmup.jl",  # a script exercising typical calls
)
# then: julia --sysimage plots_sys.so   (first plot is now near-instant)

# 2. Standalone app: MyApp must define julia_main()::Cint as entry point
create_app("path/to/MyApp", "MyAppCompiled"; filter_stdlibs = true)
# ships MyAppCompiled/bin/MyApp + bundled libjulia — no Julia install needed
```

## Architecture / How It Works

`create_sysimage` orchestrates child Julia processes rather than doing
compilation in-process. The pipeline: (1) spawn Julia with `--trace-compile`
and run your `precompile_execution_file`, recording every method
specialization the workload triggers as precompile statements; (2) spawn a
second Julia with `--output-o`, load the target packages, execute the
recorded precompile statements, and emit a native object file containing
Base, the stdlibs, your packages, and the compiled specializations; (3) link
that object file with the system C compiler into a shared library
(`.so`/`.dylib`/`.dll`) that `julia --sysimage` maps at startup[^5].

`create_app` builds a sysimage the same way, then assembles a relocatable
bundle: a thin executable wrapper that initializes the runtime and calls your
package's `julia_main()::Cint`, plus `libjulia` and its dependencies, the
sysimage, and package artifacts. `create_library` swaps the executable for
`init_julia`/`shutdown_julia` C entry points and headers around your
`@ccallable` functions. In every case the full Julia runtime — GC, dynamic
dispatch machinery, and LLVM (needed because compiled Julia can still reach
`eval`) — ships inside the artifact. That is why a hello-world app weighs
hundreds of megabytes: this is runtime bundling, not static compilation.
`filter_stdlibs = true` drops unused standard libraries and helps at the
margin, with the risk of runtime errors if a dependency loads a filtered
stdlib lazily.

The coupling story: PackageCompiler leans on unstable `julia` CLI internals
(`--output-o`, `--trace-compile`, sysimage linking details), so each Julia
minor release can require a corresponding PackageCompiler release. Version
compatibility between the tool and the Julia doing the building matters more
than for a normal package.

## Production Notes

- **Frozen code is the #1 footgun.** Packages in a sysimage override the
  active project's versions; `Pkg.update` appears to succeed but the old
  code keeps running until you rebuild the image. Revise.jl cannot track
  sysimage-baked methods. Keep sysimages out of day-to-day dev environments;
  scope them to deployment and CI.
- **Build times are long.** Rebuilding Base + stdlibs + packages takes
  minutes to tens of minutes and is CPU/RAM hungry. In CI, cache aggressively
  and rebuild only when the manifest changes.
- **No cross-compilation.** Artifacts must be built on the target OS and
  architecture. Linux relocatability follows the usual glibc rule: build on
  the oldest distro you intend to support. On macOS, bundled dylibs collide
  with signing/notarization workflows and need re-signing.
- **Precompile coverage is workload-defined.** Only specializations your
  `precompile_execution_file` actually exercises get compiled; untouched code
  paths still JIT at runtime with full first-call latency. Coverage gaps show
  up as mysteriously slow "compiled" apps.
- **Relocatability of artifacts.** Packages using lazy artifacts or
  build-time absolute paths can break when the bundle moves machines;
  `include_lazy_artifacts` and per-package auditing are the workarounds.
- **Embedding constraints.** A `create_library` product hosts one Julia
  runtime per process, cannot be cleanly unloaded, and needs care GC-rooting
  values that cross the C boundary.
- **Check whether you still need it.** On Julia 1.9+ with PrecompileTools,
  package latency is largely solved without sysimages[^3]. Reach for
  PackageCompiler when the requirement is distribution, not warm-up.

## When to Use / When Not

**Use when:**
- You ship Julia code to machines without Julia (CLI tools, on-prem apps).
- You need Julia callable from a C/C++/Fortran host process.
- CI or clusters pay Julia's startup/compile tax on every job and a
  build-once sysimage amortizes it.
- You need deterministic, frozen deployments where "the code cannot drift
  from what was built" is a feature.

**Avoid when:**
- Your problem is interactive latency on Julia 1.9+ — PrecompileTools
  workloads in the packages themselves are the modern fix[^3].
- You need small binaries: hundreds of MB is the floor here.
- Your code and dependencies change frequently — rebuild cost dominates.
- You need cross-compilation or fully static single-file executables.

## Alternatives

- JuliaLang/PrecompileTools.jl — use instead when the goal is cutting package
  load/first-call latency on Julia 1.9+, without freezing anything.
- tshort/StaticCompiler.jl — use for genuinely small standalone binaries, if
  your code fits a restricted no-runtime subset (no GC, no dynamic dispatch).
- JuliaLang/julia — the experimental `juliac` driver (1.12+) targets trimmed
  static binaries and is the likely long-term successor for apps[^4].
- timholy/SnoopCompile.jl — not a replacement; use alongside to diagnose what
  your precompile workload misses.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017-11 | Original PackageCompiler (Simon Danisch); repo created. |
| 1.0 | 2020 | Rewrite by Kristoffer Carlsson (ex-PackageCompilerX): `create_sysimage` / `create_app` API[^2]. |
| 2.0 | 2021 | Breaking release; added `create_library`, revised sysimage/app defaults[^6]. |
| — | 2023-05 | Julia 1.9 pkgimages ship; sysimage-for-latency use case shrinks[^3]. |
| — | 2025-10 | Julia 1.12 ships experimental `juliac`; PackageCompiler remains the stable path[^4]. |

## References

[^1]: PackageCompiler README — three stated purposes: sysimages, apps, C library bundles. https://github.com/JuliaLang/PackageCompiler.jl
[^2]: PackageCompiler documentation (project history and API reference). https://julialang.github.io/PackageCompiler.jl/dev/
[^3]: JuliaLang blog, "Julia 1.9 Highlights" — native code caching (pkgimages). https://julialang.org/blog/2023/04/julia-1.9-highlights/
[^4]: JuliaLang/julia NEWS — experimental `juliac` static compilation driver. https://github.com/JuliaLang/julia/blob/master/NEWS.md
[^5]: PackageCompiler docs, "Creating a sysimage". https://julialang.github.io/PackageCompiler.jl/dev/sysimages/
[^6]: PackageCompiler docs, "Upgrading from PackageCompiler 1.0". https://julialang.github.io/PackageCompiler.jl/dev/#Upgrading-from-PackageCompiler-1.0.

## Tags

julia, aot-compilation, sysimage, static-binaries, deployment, build-tooling, ffi, latency, packaging
