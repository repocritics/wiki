# Reference-LAPACK/lapack

> The reference implementation of LAPACK — the Fortran linear algebra library
> that nearly every scientific computing stack ultimately calls into.

[GitHub repo](https://github.com/Reference-LAPACK/lapack) ·
[Official website](https://www.netlib.org/lapack/) ·
[License: BSD-3-Clause (modified BSD)](https://github.com/Reference-LAPACK/lapack/blob/master/LICENSE)

## Overview

LAPACK (Linear Algebra PACKage) is a Fortran library for dense numerical
linear algebra: linear systems, least squares, eigenvalue and singular value
problems, and the factorizations (LU, Cholesky, QR, Schur) beneath them.
Version 1.0 shipped on February 29, 1992 as the successor to LINPACK and
EISPACK, redesigned around blocked algorithms that spend their time in
Level 3 BLAS matrix-matrix operations instead of memory-bound vector code[^1].
This repository is the *reference* implementation, moved to GitHub in 2016
from its long-time home at netlib; releases are still mirrored there[^2].

The ~1.9k star count understates its position by orders of magnitude: LAPACK
is what NumPy/SciPy, R, MATLAB, Julia, Octave, and Armadillo call for dense
linear algebra, and it ships in essentially every Linux distribution. The
defining tension is in the word "reference": this repo defines the algorithms
and API, but its bundled reference BLAS is deliberately naive — production
performance comes from linking against an optimized BLAS (OpenBLAS, Intel
MKL, BLIS, Apple Accelerate). Maintenance is active (pushed July 2026; 133
open issues) but conservative: the API has been stable for three decades and
releases arrive roughly yearly.

## Getting Started

Most users should install a packaged build (`apt install liblapack-dev`,
`brew install lapack`, or an optimized superset like OpenBLAS). To build the
reference implementation from source:

```sh
git clone https://github.com/Reference-LAPACK/lapack.git
cd lapack && mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=$HOME/.local/lapack -DLAPACKE=ON ..
cmake --build . -j --target install
```

Solving a linear system Ax = b from C via LAPACKE, the bundled C interface:

```c
#include <lapacke.h>

double a[9] = {2, 1, 1,  1, 3, 2,  1, 0, 0};   /* 3x3, row-major */
double b[3] = {4, 5, 6};
lapack_int ipiv[3];

lapack_int info = LAPACKE_dgesv(LAPACK_ROW_MAJOR, 3, 1, a, 3, ipiv, b, 1);
/* info == 0: b now holds x. info > 0: U is singular at that pivot. */
```

## Architecture / How It Works

The library is organized in three layers: **driver routines** (solve a whole
problem, e.g. `DGESV`), **computational routines** (one step, e.g. `DGETRF`
factorize + `DGETRS` solve), and **auxiliary routines**. Names encode
precision–matrix type–operation in 6 characters: `D`+`GE`+`SV` = double
precision, general matrix, solve. Precisions are S/D/C/Z (single, double,
complex, double complex); common matrix types are GE (general), SY/HE
(symmetric/Hermitian), PO (positive definite), GB (banded), TR (triangular).
The naming scheme is terse but fully systematic, and every downstream wrapper
(SciPy's `getrf`, Julia's `getrf!`) inherits it.

Performance strategy: partition matrices into blocks so the hot loops are
Level 3 BLAS calls (`DGEMM`, `DTRSM`), which optimized BLAS libraries tune
per-microarchitecture. LAPACK itself contains almost no architecture-specific
code — the whole design delegates machine tuning to the BLAS boundary. This is
why the same 1992-vintage Fortran remains competitive on 2026 hardware when
linked against a modern BLAS, and why the reference build without one is often
an order of magnitude slower.

Conventions that leak into every consumer: column-major storage (LAPACKE can
transpose for row-major callers, at a copy cost); error reporting via an
`INFO` output argument (negative = bad argument, positive = numerical failure
such as a singular pivot); and the workspace-query idiom — call with
`LWORK = -1` to learn the optimal scratch size, allocate, call again. The
codebase has required a Fortran 90 compiler since 3.2 (2008)[^3]; the repo
also bundles reference BLAS, CBLAS, and LAPACKE, a standard C API born from a
LAPACK/Intel MKL collaboration[^2].

## Production Notes

- **Never benchmark or deploy against the reference BLAS.** It is a
  correctness baseline. On Debian/Ubuntu, `update-alternatives` switches the
  BLAS/LAPACK provider system-wide — your Python stack's performance can
  change because of an unrelated package install.
- **32- vs 64-bit integers (LP64 vs ILP64).** Default builds use 32-bit
  `INTEGER`, capping dimensions near 2^31. ILP64 builds exist but are
  ABI-incompatible: mixing conventions between LAPACK, BLAS, and consumers
  corrupts arguments and crashes at a distance. `_64` symbol suffixing
  mitigates this but is not universal across providers.
- **Fortran ABI hazards at the C boundary.** Name mangling, hidden
  string-length arguments, and complex-return conventions differ across
  compilers. Use LAPACKE or your language's vetted bindings rather than
  hand-declared `extern` prototypes.
- **Thread safety, not parallelism.** LAPACK routines are thread-safe when the
  underlying BLAS is, but LAPACK itself is sequential — parallelism comes from
  the BLAS's internal threading. Nesting an OpenBLAS with its own thread pool
  inside your application's thread pool oversubscribes cores; set
  `OPENBLAS_NUM_THREADS`/`OMP_NUM_THREADS` deliberately.
- **Vendor drift.** Apple Accelerate long tracked an old LAPACK version; MKL
  and OpenBLAS occasionally diverge from reference in edge-case `INFO`
  behavior. Test against the implementation you actually ship.
- **Deprecations are real but glacial.** 3.6.0 (2015) deprecated legacy
  routines such as `xGEQPF` and `xGELSX`[^4]; they still build behind a flag,
  but new code should avoid them.

## When to Use / When Not

**Use when:**
- You need dense linear algebra with three decades of numerical vetting, via
  any language's bindings.
- You are building a numerical runtime and want the de facto standard API so
  users can swap in MKL/OpenBLAS/AOCL underneath.
- You need a portable, dependency-free correctness baseline to test an
  optimized implementation against.

**Avoid when:**
- Your matrices are large and sparse — use SuiteSparse, PETSc, or an iterative
  solver; LAPACK stores and factorizes dense.
- You want GPU execution — LAPACK is CPU-only; MAGMA and cuSOLVER cover that.
- You want an ergonomic API — raw LAPACK is 6-letter names, output arguments,
  and manual workspaces. Use Eigen, NumPy, or Julia and let them call LAPACK.
- Matrices are tiny (< ~16x16) and in an inner loop — call overhead dominates;
  batched/fixed-size libraries (Eigen, batched BLAS) win.

## Alternatives

- OpenMathLib/OpenBLAS — optimized BLAS plus bundled LAPACK; use when you want
  one library providing the full stack at production speed.
- flame/blis — BLAS-layer alternative, cleaner framework for new
  architectures; pair with this LAPACK on unusual hardware.
- libeigen/eigen — header-only C++ templates; use for expression-level
  ergonomics and small fixed-size matrices without Fortran.
- icl-utk-edu/magma — LAPACK-style dense algebra on GPUs (CUDA/HIP).
- scipy/scipy — in Python, its `linalg` module *is* LAPACK with memory
  management and sane names handled for you.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 1992-02-29 | Initial release; blocked successor to LINPACK/EISPACK[^1]. |
| 2.0 | 1994-09-30 | Coverage expansion across drivers. |
| 3.0 | 1999-06-30 | Major release; divide-and-conquer SVD (`xGESDD`). |
| 3.2 | 2008-11 | Fortran 90 required; extra-precise iterative refinement[^3]. |
| 3.6.0 | 2015-11 | First deprecations of legacy routines[^4]. |
| 3.7.0 | 2016-12 | First release of the GitHub development era. |
| 3.9.0 | 2019-11 | New QR-preconditioned SVD (`xGESVDQ`). |
| 3.12.0 | 2023-11 | Maintenance; continued C/CMake interface polish. |
| 3.12.1 | 2025-01-08 | Bug-fix release; latest as of writing[^5]. |

## References

[^1]: Anderson, Bai, Bischof, Demmel, Dongarra et al., *LAPACK Users' Guide*, 3rd ed., SIAM. https://www.netlib.org/lapack/lug/
[^2]: LAPACK README, Reference-LAPACK/lapack. https://github.com/Reference-LAPACK/lapack#readme
[^3]: LAPACK 3.2 release notes (Fortran 90, extended-precision refinement). https://www.netlib.org/lapack/lapack-3.2.html
[^4]: LAPACK 3.6.0 release notes (deprecated routines list). https://www.netlib.org/lapack/lapack-3.6.0.html
[^5]: LAPACK releases on GitHub. https://github.com/Reference-LAPACK/lapack/releases

## Tags

fortran, linear-algebra, blas, matrix-factorization, eigenvalues, svd, numerical-computing, scientific-computing, hpc, c-bindings
