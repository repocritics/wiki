# vlang/vsl

> The V Scientific Library — BLAS/LAPACK, numerical methods, ML primitives, and
> optional GPU backends for the V language.

[GitHub repo](https://github.com/vlang/vsl) ·
[Official website](https://vlang.github.io/vsl) ·
[License: MIT](https://github.com/vlang/vsl/blob/main/LICENSE)

## Overview

VSL is the scientific computing library of the V language ecosystem, hosted
under the official `vlang` organization. It covers linear algebra (matrix and
vector types, solvers, eigenvalue decomposition), numerical methods
(differentiation, integration, root finding), FFT, statistics and probability
distributions, K-means/KNN primitives, a Plotly-style plotting API, and
parallel computing via MPI and OpenCL[^1]. The repository dates to December
2019, started by Ulises Jeremias Cornejo Fandos and drawing its module layout
from Gosl, the Go scientific library[^2].

The defining tension is the host language. V is pre-1.0 and evolves quickly, so
VSL's audience is effectively the V community itself — at ~400 stars and 49
forks it is the reference scientific stack for a niche language, not a
contender to NumPy or Julia. What it offers that larger stacks do not: a single
compiled binary with no Python runtime, dependency-free pure-V BLAS/LAPACK
implementations, and opt-in acceleration through OpenBLAS/LAPACKE, OpenCL,
Vulkan, and CUDA[^1].

Since the v0.2.0-beta.1 "ML Beta" release (June 2026), VSL is the
compute-primitive layer beneath VTL, the V tensor/autograd library: VSL owns
GEMM, activations, softmax, LayerNorm and backend dispatch; VTL owns tensors,
layers, optimizers, and training loops[^3][^4].

## Getting Started

```sh
v install vsl
```

```v
import vsl.la

fn main() {
	mut a := la.Matrix.new[f64](2, 2)
	a.set(0, 0, 1.0)
	a.set(1, 1, 2.0)
	println(a.get(1, 1)) // 2.0
}
```

Optional accelerated backends are selected with compile flags:

```sh
v -d vsl_blas_cblas -d vsl_lapack_lapacke run main.v   # OpenBLAS/LAPACKE
v -d cuda run main.v                                    # cuBLAS/cuDNN
v -d vulkan run main.v                                  # Vulkan compute
```

## Architecture / How It Works

VSL is a collection of V modules under one repo: `la` (matrices/vectors and
solvers), `blas` and `lapack` (routine-level APIs), `ml`, `fft`, `plot`, `vcl`
(the OpenCL wrapper, "V Computing Language"), `mpi`, plus smaller numeric
modules. The architectural core is **backend layering**:

- **Pure V (default)** — dependency-free implementations of BLAS/LAPACK-style
  routines (`gemm`, `gemv`, elementwise ops, softmax, LayerNorm). Portable
  everywhere V compiles; this is the supported "beta" path for downstream
  libraries[^1].
- **C backends** — `-d vsl_blas_cblas` and `-d vsl_lapack_lapacke` route the
  same calls to system OpenBLAS/LAPACKE for optimized CPU kernels.
- **GPU backends** — OpenCL (`vcl`), Vulkan (`-d vulkan`, including a fused
  Adam-step shader and Conv2D im2col), and CUDA (`-d cuda`, cuBLAS/cuDNN GEMM,
  activations, Conv2D). All are explicitly opt-in and marked as early-adopter
  paths, not required for the default CPU story[^1].

A backend-agnostic dispatch layer, `vsl.compute`, is the recommended
integration point for downstream code; per-backend implementations live in
`vsl/vcl/compute`, `vsl/vulkan/compute`, and `vsl/cuda/compute`[^1]. This
mirrors the NumPy/BLAS split — a stable routine surface over swappable
kernels — but backend selection is a compile flag, visible in every build.

## Production Notes

- **V itself is the biggest risk.** V is pre-1.0 with frequent breaking
  changes; VSL tracks the moving language, and `v install vsl` fetches from
  the repository head rather than a pinned artifact. Pin both V and VSL
  commits in CI.
- **Known correctness gap:** the pure-V QR path (`geqrf`/`orgqr`) is under
  alignment and its test is skipped; the README itself recommends the C
  backends when QR correctness matters[^1]. Treat pure-V LAPACK coverage as
  partial and verify the routines you depend on.
- **Release cadence is bursty.** Tags ran v0.1.44–v0.1.50 through 2022–2023,
  then nothing until v0.1.51 in December 2025 — a near-three-year gap during
  which development continued on `main`[^6]. Activity has resumed (last push
  July 2026), but plan around git commits, not versions.
- **Compile-time cost of scope:** the repo's own docs advise scoped testing
  (`v test vsl/blas vsl/la vsl/compute`) — compiling the whole stack is slow[^1].
- **Bus factor:** development concentrates in a small group around the
  original author; 37 open issues against a small team means bug reports sit.
- **C backend setup is on you** — OpenBLAS/LAPACKE, OpenMPI, OpenCL, and HDF5
  are system dependencies with per-module compilation flags. The Docker
  starter template (`ulises-jeremias/hello-vsl`) ships a known-good
  environment[^5].

## When to Use / When Not

**Use when:**
- You are already building in V and need linear algebra, statistics, FFT, or
  plotting without leaving the language.
- You want small static binaries for numeric tooling with no Python/Julia
  runtime, and pure-V portability matters more than peak FLOPS.
- You are experimenting with VTL for ML in V — VSL is its required compute
  layer[^4].

**Avoid when:**
- You need a battle-tested numerical stack for production science — NumPy/
  SciPy, Julia, or direct OpenBLAS bindings have decades more validation.
- You need stable versioned releases and long-term API guarantees; both V and
  VSL are moving targets.
- Your workload is GPU-first deep learning at scale — the CUDA/Vulkan backends
  are early-adopter paths, not a PyTorch substitute[^1].

## Alternatives

- numpy/numpy — use for any Python-adjacent workflow; vastly larger ecosystem.
- JuliaLang/julia — use when scientific computing is the project's center of
  gravity rather than a module in a V app.
- cpmech/gosl — the Go library VSL's design descends from; same scope in a
  stable 1.0 language.
- OpenMathLib/OpenBLAS — use directly via FFI when you only need fast BLAS
  kernels without a library layer.
- vlang/vtl — not a substitute but the companion: tensors/autograd/NN on top of
  VSL's primitives.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-12 | Repository created; design modeled on Gosl[^2]. |
| v0.1.46 | 2022-08 | Docker-based release workflow. |
| v0.1.47 | 2022-10 | Full test suite passing under `-prod`. |
| v0.1.50 | 2023-02 | VCL (OpenCL) image support working. |
| v0.1.51 | 2025-12 | Pure-V BLAS/LAPACK benchmarks and examples; first tag after ~34-month gap[^6]. |
| v0.2.0-beta.1 | 2026-06 | "ML Beta": `vsl.compute` backend standardization, CUDA/Vulkan backends, VTL split[^3]. |

## References

[^1]: VSL README — capabilities, backend matrix, QR caveat. https://github.com/vlang/vsl#readme
[^2]: Gosl — Go scientific library (cpmech/gosl). https://github.com/cpmech/gosl
[^3]: VSL release v0.2.0-beta.1 — ML Beta, 2026-06-02. https://github.com/vlang/vsl/releases/tag/v0.2.0-beta.1
[^4]: VTL — V Tensor Library (tensors, autograd, NN layers). https://github.com/vlang/vtl
[^5]: hello-vsl — Docker starter template. https://github.com/ulises-jeremias/hello-vsl
[^6]: VSL release v0.1.51 — 2025-12-28. https://github.com/vlang/vsl/releases/tag/v0.1.51

## Tags

v, scientific-computing, linear-algebra, blas, lapack, numerical-methods, machine-learning, gpu-acceleration, opencl, cuda, vulkan, fft
