# JuliaGPU/CUDA.jl

> CUDA programming in Julia — high-level GPU arrays and hand-written kernels in the same language, compiled by the same compiler.

[GitHub repo](https://github.com/JuliaGPU/CUDA.jl) ·
[Official website](https://juliagpu.org/backends/cuda/) ·
[License: MIT](https://github.com/JuliaGPU/CUDA.jl/blob/main/LICENSE.md)

## Overview

CUDA.jl is the main interface for programming NVIDIA GPUs from Julia. Its unusual promise is that both levels of GPU programming live in one language: `CuArray` gives a drop-in array type where broadcasts, `map`, and reductions compile to fused GPU kernels, and the `@cuda` macro compiles ordinary Julia functions to PTX for hand-written kernels[^1]. There is no C++ shim layer and no separate kernel language — the same Julia compiler that produces CPU code emits GPU code through LLVM's NVPTX backend.

The package consolidated three predecessors (CUDAdrv.jl, CUDAnative.jl, CuArrays.jl) into a single package with its 1.0 release in June 2020[^2]. Since v6.0 (April 2026) it is a meta-package importing registered subpackages: CUDACore (API wrappers + kernel compiler), CUDATools (profiler, CUPTI, NVML), and wrappers for cuBLAS, cuFFT, cuRAND, cuSPARSE, and cuSOLVER; cuTENSOR, cuDNN, cuStateVec, and cuTensorNet are opt-in extras[^3]. All subpackages version in lockstep.

At ~1.4k stars it looks small next to Python GPU stacks, but that undersells its position: it is the de facto standard for NVIDIA GPU computing in Julia, the substrate under Flux.jl, Lux.jl, and the SciML ecosystem's GPU support. Development is active — the latest release (v6.2.1) shipped in July 2026 and the repo sees near-daily pushes. The defining tension: Julia's dynamism (dynamic dispatch, GC, exceptions) meets a device that supports none of it, and much of the package's design is about managing that boundary.

## Getting Started

No CUDA Toolkit install is needed — CUDA.jl downloads a toolkit matching your driver via Julia's artifact system. You only need a recent NVIDIA driver (CUDA 12+), 64-bit Linux or Windows, and Julia 1.10+[^4].

```julia
julia> import Pkg; Pkg.add("CUDA")
julia> using CUDA
julia> CUDA.versioninfo()   # first run downloads a toolkit
```

Array-level and kernel-level programming:

```julia
using CUDA

a = CUDA.rand(Float32, 1024)
b = CUDA.rand(Float32, 1024)
c = a .+ 2f0 .* b        # broadcast fuses into a single GPU kernel
sum(c)                   # GPU reduction

# Hand-written kernel: plain Julia, compiled to PTX by @cuda
function vadd!(c, a, b)
    i = (blockIdx().x - 1) * blockDim().x + threadIdx().x
    if i <= length(c)
        @inbounds c[i] = a[i] + b[i]
    end
    return nothing
end

@cuda threads=256 blocks=cld(length(c), 256) vadd!(c, a, b)
```

A `ghcr.io/juliagpu/cuda.jl` container image ships Julia, precompiled CUDA.jl, and a matching toolkit, tagged by CUDA major version (`cuda12`, `cuda13`)[^3].

## Architecture / How It Works

The kernel path runs through **GPUCompiler.jl**: it reuses Julia's own type inference and optimization pipeline, lowers a kernel function to LLVM IR with GPU-specific intrinsics, and emits PTX via LLVM's NVPTX backend. Compilation happens at first launch and is cached per method/type signature. Because it is the real Julia compiler, kernels get multiple dispatch, parametric types, and metaprogramming — but only the statically resolvable subset: kernels must be type-stable, cannot allocate on the GC heap, cannot fall back to runtime dynamic dispatch, and exceptions reduce to a device-side trap with limited diagnostics.

The array path builds `CuArray` on **GPUArrays.jl**, a backend-agnostic interface layer. Broadcasting uses Julia's broadcast machinery, so `a .+ 2 .* b` becomes one generated kernel rather than three temporaries — kernel fusion falls out of the language semantics instead of a graph compiler.

Other load-bearing internals:

- **Memory management** — allocations go through CUDA's stream-ordered allocator (`cudaMallocAsync`) behind a pool. `CuArray`s are freed by Julia's GC, which cannot see GPU memory pressure; on allocation failure CUDA.jl triggers GC and retries before surfacing an OOM.
- **Task/stream integration** — each Julia task gets its own CUDA stream, so concurrent Julia tasks map to concurrent GPU work without manual stream plumbing (introduced in v3.0)[^5].
- **Toolkit artifacts** — the CUDA runtime ships as versioned JLL artifact packages; `CUDA.set_runtime_version!` pins a version or opts into a system toolkit, persisted via `LocalPreferences.toml`[^3].
- **Scalar-indexing guard** — falling back to CPU-style `arr[i]` loops on a `CuArray` would issue one kernel launch per element; CUDA.jl errors on this in non-interactive code unless explicitly allowed with `CUDA.@allowscalar`.

## Production Notes

**Time-to-first-kernel.** Kernels compile at first launch; loading the package plus compiling a nontrivial kernel takes seconds. Julia 1.9+ native code caching improved package load times, but GPU code generation remains a runtime cost. Short-lived batch jobs pay it on every run; long-running services amortize it.

**GPU OOM behaves oddly.** Because the GC owns `CuArray` lifetimes, memory can look "leaked" while unreferenced arrays await collection in the pool. The allocator GCs and retries under pressure, which shows up as mysterious pauses. `CUDA.reclaim()` releases pooled memory, and a hard memory limit can be configured for shared GPUs. Multi-process GPU sharing (one GPU, several Julia workers) needs this discipline or processes starve each other.

**Scalar indexing is the #1 newcomer footgun.** Generic library code that iterates elementwise will either error (scripts) or crawl at thousands of kernel launches per second. The fix is vectorizing with broadcast/`map` or writing a kernel — not `@allowscalar`, which is a diagnostic escape hatch.

**Kernel restrictions surface as compiler errors.** Type instability or hidden allocation inside a kernel fails at `@cuda` time with GPUCompiler errors that are markedly harder to read than ordinary Julia stack traces. `@device_code_warntype` and Cthulhu.jl are the standard diagnosis tools.

**Version ladders matter.** CUDA.jl v5.9 dropped CUDA 11 and Kepler GPUs; current releases require compute capability 5.0+ (Maxwell) and a CUDA 12+ driver. Older hardware needs pinned older versions (v5.8 for CUDA 11, v4.4 for CUDA 11.0–11.3)[^4]. Air-gapped clusters must pre-mirror the multi-gigabyte toolkit artifacts or use `set_runtime_version!(local_toolkit=true)`.

**No CPU fallback.** Code written directly against CUDA.jl requires an NVIDIA GPU, including in CI (the project itself runs GPU CI on Buildkite). Libraries wanting portability write kernels against KernelAbstractions.jl and treat CUDA.jl as one backend.

## When to Use / When Not

**Use when:**
- You are in Julia and need NVIDIA GPU acceleration — this is the standard path, not one option among many.
- Your workload fits array semantics: broadcasts and reductions get you fused GPU kernels with near-zero code changes.
- You need custom kernels but want one language with real metaprogramming instead of C++/CUDA plus a binding layer.
- You use the vendor libraries (cuBLAS, cuFFT, cuSPARSE, cuSOLVER, cuDNN) and want idiomatic wrappers.

**Avoid when:**
- Your stack is Python — CuPy, Numba, JAX, or PyTorch have larger ecosystems and you gain nothing crossing languages.
- You need AMD, Intel, or Apple GPUs — CUDA.jl is NVIDIA-only by design.
- Deployment targets lack NVIDIA drivers, or startup latency (first-kernel compilation) dominates short-lived jobs.
- You cannot tolerate GC-mediated GPU memory lifetimes in tightly memory-constrained multi-tenant GPU setups.

## Alternatives

- JuliaGPU/AMDGPU.jl — same programming model for AMD ROCm GPUs.
- JuliaGPU/Metal.jl — Julia GPU programming on Apple silicon.
- JuliaGPU/oneAPI.jl — Intel GPUs via the oneAPI stack.
- JuliaGPU/KernelAbstractions.jl — write vendor-portable kernels once, run on CUDA.jl, AMDGPU.jl, or CPU; use it for library code that must not hard-depend on NVIDIA.
- cupy/cupy — NumPy-compatible GPU arrays; use it when your codebase is Python, not Julia.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2020-06-17 | Merger of CUDAdrv.jl + CUDAnative.jl + CuArrays.jl into one package[^2]. |
| 2.0 | 2020-10-02 | Second major release. |
| 3.0 | 2021-04-09 | Task-based concurrency (stream per Julia task), stream-ordered allocator[^5]. |
| 4.0 | 2023-02-01 | CUDA toolkit delivered via JLL artifact packages. |
| 5.0 | 2023-09-19 | Deprecated CUDA 11.0–11.3; CUDA 12 era[^4]. |
| 6.0 | 2026-04-10 | Meta-package split into CUDACore, CUDATools, and per-library subpackages[^3]. |
| 6.2.1 | 2026-07-09 | Latest release as of this page. |

## References

[^1]: T. Besard, C. Foket, B. De Sutter, "Effective Extensible Programming: Unleashing Julia on GPUs", IEEE TPDS, 2018. https://doi.org/10.1109/TPDS.2018.2872064
[^2]: CUDA.jl v1.0.0 release. https://github.com/JuliaGPU/CUDA.jl/releases/tag/v1.0.0
[^3]: CUDA.jl README — organization, containers, toolkit selection. https://github.com/JuliaGPU/CUDA.jl#readme
[^4]: CUDA.jl README — requirements and legacy version support table. https://github.com/JuliaGPU/CUDA.jl#requirements
[^5]: CUDA.jl v3.0.0 release notes. https://github.com/JuliaGPU/CUDA.jl/releases/tag/v3.0.0

## Tags

julia, gpu, cuda, nvidia, gpgpu, scientific-computing, high-performance-computing, compiler, array-programming, numerical-computing
