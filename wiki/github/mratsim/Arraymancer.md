# mratsim/Arraymancer

> A NumPy/PyTorch-inspired tensor library for Nim, with CPU, CUDA, and OpenCL backends and a macro-based neural-network DSL.

[GitHub repo](https://github.com/mratsim/Arraymancer) ·
[Official docs](https://mratsim.github.io/Arraymancer/) ·
[License: Apache-2.0](https://github.com/mratsim/Arraymancer/blob/master/LICENSE)

## Overview

Arraymancer is an n-dimensional array (tensor) library written in Nim, started
by Mamy Ratsimbazafy in April 2017. It aims to be the foundation of a
scientific computing ecosystem in Nim: the core is a NumPy-style ndarray with
slicing, broadcasting, and BLAS/LAPACK-backed linear algebra; on top of that
sit scikit-learn-style algorithms (least squares, PCA, k-means) and a
PyTorch-style define-by-run autograd with a declarative neural-network DSL[^1].
At roughly 1.4k stars it is one of the most visible scientific libraries in
the Nim ecosystem — which is to say, small in absolute terms.

The pitch is Nim itself: Python-like syntax at C-like speed, ~5-second
compiles that make the edit-run loop tolerable without a REPL, and deployment
as a nearly dependency-free native binary — the README frames this against
NumPy's C/Cython two-language problem and Python DL's research-to-production
gap[^1]. The defining tension is ambition versus bus factor: the library
covers tensors, ML, and DL across three compute backends, yet still carries a
"stability: experimental" badge, declares the deep-learning interface
work-in-progress, and has shipped no minor release since v0.7.0 in July
2021[^2]. Patch releases (v0.7.33, October 2024) and master commits (last
push May 2026) continue — maintenance-mode velocity, not roadmap velocity.

## Getting Started

```bash
nimble install arraymancer   # requires Nim (install via choosenim)
```

A BLAS and LAPACK library must be present: Apple Accelerate covers macOS,
`libopenblas`/`liblapack` via package manager on Linux; Windows needs
`libopenblas.dll` manually placed on the path[^1].

```nim
import arraymancer

let a = [[1.0, 2.0],
         [3.0, 4.0]].toTensor
let b = [[5.0, 6.0],
         [7.0, 8.0]].toTensor

echo a * b        # matrix multiplication (BLAS-backed)
echo a +. b       # element-wise addition — broadcasting ops are dot-suffixed
echo a[_, 0]      # slicing: first column
```

Compile with `-d:release` (or `-d:danger` to also drop bounds checks),
`-d:openmp` for multithreading, `-d:cuda` / `-d:opencl` for GPU backends[^1].

## Architecture / How It Works

- **Tensor core.** A `Tensor[T]` is shape/stride metadata over a data buffer;
  slicing with ranges, steps, and negative indexing (`foo[_|-1, _]` reverses
  rows) is expressed through Nim's operator overloading and macros. Tensors
  support up to 6 dimensions. CPU tensors are generic over element type —
  integers, strings, booleans, custom objects — not just floats[^1].
- **Explicit broadcasting.** Unlike NumPy, broadcasting never happens
  implicitly: element-wise broadcasted operators are spelled `+.`, `-.`, `*.`
  etc., while bare `*` on 2-D tensors is matrix multiplication. This trades
  NumPy muscle memory for eliminating a whole class of silent shape bugs.
- **Backend selection at compile time.** BLAS/LAPACK are resolved through the
  `nimblas`/`nimlapack` shims, overridable with `-d:blas=...` /
  `-d:lapack=...`[^1]. CUDA and OpenCL tensors are distinct types
  (`CudaTensor`, `ClTensor`) created by converting a CPU tensor; they are
  restricted to `float32`/`float64` and implement a subset of the CPU feature
  matrix — the README maintains an honest comparison table[^1].
- **Neural-network DSL.** A `network` macro declares layers and a `forward`
  expression; it expands to a model type, initializer, and forward proc.
  Autograd is define-by-run: a `Context` records operations on `Variable`s,
  `loss.backprop()` walks the graph, and optimizers (`SGD`, etc.) update
  in place — structurally a small PyTorch, at compile-time-typed Nim prices.
- **Native output.** Nim compiles to C/C++, so the artifact is a native
  binary; the same code targets embedded/IoT. The planned next-generation
  compute backend, `laser`, was split into its own repository and never fully
  merged back[^3] — CPU kernels remain BLAS calls plus hand-written OpenMP
  loops.

## Production Notes

- **Experimental means experimental.** The stability badge is not false
  modesty: the NN interface is explicitly subject to change, and the API docs
  are only generated for 0.x releases while devel evolutions live in the
  examples folder[^1]. Pin your version.
- **Effectively single-maintainer, in maintenance mode.** The author's focus
  visibly moved to other projects (the `laser`/`weave` HPC line, later
  cryptography)[^3]; recent activity is community patches and Nim-version
  compatibility fixes. 165 open issues against ~1.4k stars is a normal ratio,
  but expect slow triage. Budget for reading source.
- **GPU backends lag far behind CPU.** `CudaTensor`/`ClTensor` lack
  single-element access and much of the CPU op surface[^1]; the autograd/NN
  stack is primarily exercised on CPU. Treat "CUDA support" as accelerated
  linear algebra, not as PyTorch-on-GPU parity.
- **BLAS discovery is a real-world footgun.** Auto-detection of
  `blas.so`/`libopenblas.dll` works on Mac/Linux defaults but Windows setups
  routinely need manual DLL placement and `nim.cfg` path tuning[^1].
- **`-d:avx512` produces CPU-specific binaries** — the flag passes
  `-mavx512dq` to the C compiler, so the output crashes on non-AVX512 hosts;
  without it, AVX512 silicon goes unused[^1].
- **No REPL.** Fast compiles soften this, but Jupyter-style exploration is
  not the workflow; Arraymancer suits programs, not notebooks.

## When to Use / When Not

**Use when:**
- You are already committed to Nim and need tensors, linear algebra, or
  classic ML without FFI to Python.
- You want numerical code deployed as a small static-ish native binary
  (CLI tools, embedded/edge targets).
- You want NumPy-style ergonomics with compile-time type checking and
  explicit broadcasting.

**Avoid when:**
- You need production deep learning — modern architectures, GPU training
  parity, and pretrained-model ecosystems live in PyTorch/JAX, not here.
- You need a stable 1.0 API or vendor-style support; nine years in, the
  library is still 0.7.x and experimental by its own labeling.
- Your team's workflow is notebook-centric exploration.
- You need >6-dimensional arrays or GPU tensors of non-float types.

## Alternatives

- numpy/numpy — use instead when you want the default scientific stack and
  ecosystem gravity outweighs Nim's deployment story.
- pytorch/pytorch — use instead for any serious deep learning; Arraymancer's
  NN layer is a structural homage, not a competitor.
- SciNim/flambeau — use instead if you want libtorch's kernels from Nim,
  accepting a heavyweight C++ dependency.
- andreaferretti/neo — use instead for plain dense linear algebra in Nim
  without the tensor/DL superstructure.
- JuliaLang/julia — use instead when you want compiled-speed numerics plus an
  interactive REPL-first workflow.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 "Magician Apprentice" | 2017-07-12 | First release; releases are codenamed after fantasy novels[^2]. |
| v0.2.0 "The Color of Magic" | 2017-09-24 | Early tensor-core era. |
| v0.4.0 "The Name of the Wind" | 2018-05-05 | |
| v0.5.0 "Sign of the Unicorn" | 2018-12-23 | |
| v0.6.0 "Windwalkers" | 2020-01-08 | |
| v0.7.0 "Memories of Ice" | 2021-07-04 | Last minor release[^2]. |
| v0.7.33 | 2024-10-11 | Latest tag; 0.7.x patch series for compatibility and fixes[^2]. |

## References

[^1]: Arraymancer README (installation, compilation flags, feature matrix, stability badge). https://github.com/mratsim/Arraymancer#readme
[^2]: Arraymancer releases and tags. https://github.com/mratsim/Arraymancer/releases
[^3]: mratsim/laser — "The HPC toolbox", started as the next-gen Arraymancer backend. https://github.com/mratsim/laser

## Tags

nim, tensor, ndarray, deep-learning, machine-learning, linear-algebra, scientific-computing, cuda, opencl, autograd, hpc
