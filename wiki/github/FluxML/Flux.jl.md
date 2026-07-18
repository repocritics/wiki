# FluxML/Flux.jl

> Julia's flagship deep-learning library: models are plain Julia functions and structs, differentiated source-to-source — no graph compiler, no C++ core.

[GitHub repo](https://github.com/FluxML/Flux.jl) ·
[Official website](https://fluxml.ai/) ·
[License: MIT](https://github.com/FluxML/Flux.jl/blob/master/LICENSE.md)

## Overview

Flux is a deep-learning library written entirely in Julia, started by Mike Innes at Julia Computing in 2016 and published as a JOSS paper in 2018[^1]. Its thesis is the inverse of the PyTorch/TensorFlow design: instead of a Python frontend driving a C++/CUDA core, use one language that is fast enough to be its own backend. Any parameterised Julia function — a closure, a callable struct — is a valid Flux model; gradients come from Zygote's source-to-source automatic differentiation of ordinary Julia code[^2]. This "differentiable programming" stance is why Flux became the neural-network layer of Julia's scientific-ML ecosystem (neural ODEs, physics-informed networks, DiffEqFlux), where models are composed with ODE solvers rather than isolated in a training script.

At ~4,700 stars it is the most visible Julia ML package — tiny next to PyTorch, but the center of gravity for deep learning in the Julia world. The repo is actively maintained (pushed July 2026; v0.16.10 released April 2026), yet after ten years it is still pre-1.0, and under Julia's semver convention every 0.x minor release may break: the version number is an honest signal that the API still moves. The defining trade is elegance versus ecosystem mass: Flux itself is a thin, hackable layer of glue, and in exchange you give up the pretrained model zoo, fused-compiler performance, and deployment tooling of the Python stack.

## Getting Started

```julia
using Pkg; Pkg.add("Flux")
```

A minimal classifier in the current (explicit-gradient) style:

```julia
using Flux

x = rand(Float32, 2, 1000)                       # features are columns
y = Flux.onehotbatch(x[1, :] .> x[2, :], [true, false])

model = Chain(Dense(2 => 16, relu), Dense(16 => 2))
opt_state = Flux.setup(Adam(0.01), model)
loader = Flux.DataLoader((x, y), batchsize=64, shuffle=true)

for epoch in 1:50, (xb, yb) in loader
    grads = Flux.gradient(m -> Flux.logitcrossentropy(m(xb), yb), model)
    Flux.update!(opt_state, model, grads[1])
end
```

GPU use is `model |> gpu` / `data |> gpu` after loading a backend package (`CUDA` + `cuDNN`, `AMDGPU`, or `Metal`).

## Architecture / How It Works

Flux is deliberately small; most of the machinery lives in sibling FluxML packages, and Flux mostly assembles them:

- **NNlib.jl** — the kernel layer: convolutions, pooling, softmax, activation functions, with CPU and GPU dispatch.
- **Zygote.jl** — reverse-mode AD that transforms Julia's SSA-form IR, so unmodified Julia code (control flow, recursion, custom structs) is differentiable without tracing or a tape[^2]. Custom gradients hook in via ChainRules.jl rules.
- **Functors.jl / `@layer`** — treats a model as a tree of nested structs and walks it to collect parameters, move devices, or map functions. There is no base-class contract like `nn.Module`; a layer is any callable struct opted in with the `@layer` macro.
- **Optimisers.jl** — optimiser rules as pure functions over an explicit state tree (`Flux.setup` / `update!`), which replaced the older implicit `Flux.params` mechanism[^3].
- **MLUtils.jl** — `DataLoader`, batching, splitting.

`Chain` is essentially function composition; `Dense` is a ~10-line struct. This is the library's real pitch: reading and modifying the source is expected practice, not spelunking.

There is no graph capture or fusion compiler. Execution is eager, one kernel per op, with performance coming from Julia's JIT and vendor libraries (cuDNN via CUDA.jl). GPU backends attach through package extensions (weak dependencies) since v0.14, so Flux itself stays lightweight[^4]. Zygote's known holes are the coupling story's weak point: array mutation is unsupported inside differentiated code, scalar indexing on GPU arrays is pathologically slow, and second-order derivatives are fragile[^5]. Recent Flux versions also support Enzyme.jl (LLVM-level AD) as an alternative gradient engine for code Zygote cannot handle.

## Production Notes

**The 0.15 break is the big one.** In December 2024 Flux removed the implicit `Flux.params`-based training API that a decade of tutorials, Stack Overflow answers, and model-zoo forks were written against[^3]. Consequence in 2026: most Flux code on the internet — and most LLM-generated Flux code — uses the dead API. Budget time for translating any example older than ~2023 to `Flux.setup`/explicit `gradient`.

**Compile latency (TTFX).** The first `gradient` call in a session compiles for seconds to tens of seconds. Julia 1.9+ native code caching and Flux precompilation improved cold start, but the workflow assumption is a long-lived session (Revise.jl, notebooks). Short-lived batch jobs pay the compile tax every run.

**Float32 discipline.** Julia numeric literals default to `Float64`; mixing them into a `Float32` model silently promotes arrays and can halve GPU throughput. Flux layers are `Float32` by default and newer versions warn on eltype mismatch, but the footgun survives in loss functions and data pipelines.

**Checkpointing is version-brittle.** Serializing whole models with BSON/JLD2 ties the file to the exact struct layout of that Flux version; upgrades break old checkpoints. The supported pattern is `Flux.state(model)` saved via JLD2, with the model rebuilt from code and `Flux.loadmodel!` applied. There is no practical ONNX export path, so deployment means running Julia or hand-porting weights.

**Scale ceiling.** Single-GPU dense/conv workloads are competitive via cuDNN, but there is no first-class distributed data-parallel story (community pieces exist around MPI.jl; nothing with the maturity of PyTorch DDP/FSDP), no `torch.compile`-style fusion, and few maintained pretrained transformers. Metalhead.jl covers standard vision architectures; beyond that expect to train from scratch.

**Maintenance reality.** Original author Mike Innes has moved on; a small maintainer group (Carlo Lucibello and others) keeps a steady release cadence. 192 open issues on a project this old is moderate, but the bus factor is visibly smaller than the Python incumbents'.

## When to Use / When Not

**Use when:**
- You are already in Julia — simulation, optimization, SciML — and need neural networks composed with that code rather than bridged over PyCall.
- Your model is genuinely nonstandard (differentiating through a solver, a physics simulator, custom control flow) and tracing-based ADs fight you.
- You want a DL stack small enough to read end-to-end and hack on.

**Avoid when:**
- You need pretrained LLMs/transformers, HuggingFace-style ecosystems, or ONNX/mobile deployment — the Python stack wins outright.
- You are training at multi-node scale; there is no mature DDP equivalent.
- Team familiarity is Python-only: you would be adopting a language, not just a library, and the 0.x churn compounds the onboarding cost.

## Alternatives

- LuxDL/Lux.jl — Julia DL with fully explicit parameters and state; preferred by the SciML ecosystem when you want deterministic, functional-style models.
- pytorch/pytorch — use instead for pretrained models, deployment tooling, and distributed training at any serious scale.
- jax-ml/jax — functional autodiff plus XLA fusion; closest in spirit to differentiable programming but with a compiler and a bigger ecosystem.
- PumasAI/SimpleChains.jl — much faster for small CPU-only MLPs; no GPU, no generality.
- denizyuret/Knet.jl — the earlier Julia DL library; effectively dormant, of historical interest only.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016 | Project starts at Julia Computing (Mike Innes)[^1]. |
| 0.10 | 2019-11 | Zygote replaces Tracker as the AD engine[^2]. |
| 0.12 | 2021-03 | CUDA.jl-based GPU stack consolidated. |
| 0.13 | 2022-04 | Explicit training path via Optimisers.jl introduced. |
| 0.14 | 2023-07 | GPU backends become package extensions; `@layer` era begins[^4]. |
| 0.15 | 2024-12 | Implicit `Flux.params` API removed; Julia ≥ 1.10 required[^3]. |
| 0.16 | 2024-12 | Follow-up breaking release, ten days after 0.15. |
| 0.16.10 | 2026-04 | Latest release at time of writing. |

## References

[^1]: Innes, "Flux: Elegant machine learning with Julia", JOSS 3(25), 2018. https://doi.org/10.21105/joss.00602
[^2]: Innes et al., "Fashionable Modelling with Flux", 2018. https://arxiv.org/abs/1811.01457 — Zygote: "Don't Unroll Adjoint", https://arxiv.org/abs/1810.07951
[^3]: Flux.jl NEWS.md (release notes, incl. v0.15 removal of implicit params). https://github.com/FluxML/Flux.jl/blob/master/NEWS.md
[^4]: Flux docs, "GPU Support". https://fluxml.ai/Flux.jl/stable/guide/gpu/
[^5]: Zygote.jl docs, "Limitations". https://fluxml.ai/Zygote.jl/stable/limitations/

## Tags

julia, deep-learning, machine-learning, neural-networks, automatic-differentiation, gpu, scientific-computing, differentiable-programming, library
