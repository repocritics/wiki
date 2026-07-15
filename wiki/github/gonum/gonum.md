# gonum/gonum

> Numeric libraries for Go — matrices, statistics, optimization, graphs — the SciPy-shaped standard library Go never shipped.

[GitHub repo](https://github.com/gonum/gonum) ·
[Official website](https://www.gonum.org) ·
[License: BSD-3-Clause](https://github.com/gonum/gonum/blob/master/LICENSE)

## Overview

Gonum is a suite of numeric packages for Go under the single module `gonum.org/v1/gonum`. It covers dense/banded/symmetric matrices (`mat`), BLAS and LAPACK abstractions (`blas`, `lapack`), descriptive and inferential statistics with probability distributions (`stat`, `stat/distuv`, `stat/distmv`), gradient and derivative-free optimization (`optimize`), numerical integration and differentiation (`integrate`, `diff/fd`), graph theory (`graph` and subpackages), FFTs (`dsp/fourier`), and low-level float slice helpers (`floats`, `mathext`). It is the closest thing the Go ecosystem has to NumPy + SciPy, though the analogy is loose: the API surface and the ergonomics are very different.

The project is the 2017 consolidation of several earlier, separately-versioned Go numeric libraries (notably the `matrix`/`mat64` and `stat` repositories) into one monorepo[^1]. It is maintained by a small volunteer core rather than a company, and it deliberately stays close to the Go standard library in style: explicit, allocation-conscious, `float64`-centric, no generics-driven abstraction over element types.

The defining tension is maturity versus stability guarantees. Gonum is old, widely depended on, and heavily tested, yet it has never shipped a 1.0 — releases are `v0.x` and the repository's own README carries a "stability: unstable" badge[^2]. The `v1` in the import path is a module vanity suffix, not a promise of API stability. In practice the core APIs (`mat`, `stat`, `floats`) change slowly, but you are pinning to a project that reserves the right to break, and does occasionally break, on minor bumps.

## Getting Started

```bash
go get gonum.org/v1/gonum/...
```

A small linear-algebra + statistics example:

```go
package main

import (
	"fmt"

	"gonum.org/v1/gonum/mat"
	"gonum.org/v1/gonum/stat"
)

func main() {
	// Solve A x = b for x.
	a := mat.NewDense(2, 2, []float64{3, 2, 1, 2})
	b := mat.NewVecDense(2, []float64{5, 5})

	var x mat.VecDense
	if err := x.SolveVec(a, b); err != nil {
		panic(err)
	}
	fmt.Printf("x = %.4v\n", mat.Formatted(&x))

	// Weighted mean and standard deviation.
	xs := []float64{2, 4, 4, 4, 5, 5, 7, 9}
	mean, std := stat.MeanStdDev(xs, nil)
	fmt.Printf("mean=%.3f std=%.3f\n", mean, std)
}
```

The API convention is that destination values receive the result: you allocate a zero `mat.Dense`/`mat.VecDense` and call a method like `Mul`, `Solve`, or `SolveVec` on it. Reusing destinations across a loop is how you avoid per-iteration allocation.

## Architecture / How It Works

Gonum is layered. At the bottom are `blas` and `lapack`, which define Go interfaces mirroring the classic Fortran BLAS/LAPACK routines. Two implementations satisfy them: a pure-Go backend (`blas/gonum`, `lapack/gonum`) that ships by default, and a cgo backend in the separate `gonum.org/v1/netlib` module that binds to a system BLAS/LAPACK (OpenBLAS, Intel MKL, reference netlib). The `blas64`/`lapack64` packages expose a registered-default indirection so you can swap the backend process-wide without touching call sites.

The `mat` package sits on top of that. `mat.Dense` and friends store elements in **row-major** order with an explicit stride, which is why sub-matrix views (`Slice`) can alias the parent's backing array. Matrix types implement small interfaces (`mat.Matrix` is just `Dims`, `At`, `T`), so custom matrix types interoperate with the package's algorithms. Decompositions (`LU`, `Cholesky`, `QR`, `SVD`, `Eigen`) are value types you factorize once and reuse.

Performance-critical inner loops in the pure-Go BLAS have hand-written amd64 assembly (AVX) with pure-Go fallbacks; build tags (`noasm`, `safe`) select fallbacks, `bounds` forces internal bounds checks, and `tomita` switches a clique-pivot heuristic in `graph/topo`[^2]. Because gonum predates Go generics (Go 1.18, 2022) and has not retrofitted them into the core, nearly everything is concrete `float64` (with `complex128` where relevant). There is no generic `Matrix[T]`; if you need `float32` or integer math you are largely on your own.

## Production Notes

**No 1.0, real break risk.** The `v0.x` line means the module is exempt from Go's semantic-import-versioning stability expectations. Minor releases have removed or changed signatures. Pin an exact version in `go.mod` and read release notes before bumping; do not treat `go get -u` as safe.

**Pure-Go BLAS is correct but not MKL-fast.** The default backend is portable and dependency-free, which is a real advantage for reproducible builds and cross-compilation. But for large dense linear algebra it is meaningfully slower than a tuned native BLAS. If matrix multiply / factorization is your hot path, wire in `gonum.org/v1/netlib` against OpenBLAS or MKL — at the cost of cgo, a C toolchain, and losing easy cross-compilation.

**Allocation discipline is on you.** The destination-receiver API rewards reuse and punishes naive code: writing `c.Mul(a, b)` with a fresh `c` each loop iteration allocates a new backing array every time. Hoist destinations out of loops. Aliasing rules matter too — some operations forbid overlapping input/output and will panic (`mat.ErrShape`/mismatch panics) rather than silently corrupt.

**Panics, not just errors.** Much of `mat` panics on dimension mismatch rather than returning an error, mirroring the standard library's slice-bounds philosophy. Numeric routines that can legitimately fail (singular solve, non-convergence) return errors. Know which is which; do not wrap the whole thing in `recover` and hope.

**Floating-point non-determinism across platforms.** The README explicitly warns that results can differ between compiler versions and architectures due to differing float implementations[^2]. Do not hard-assert bit-exact numeric outputs in cross-platform tests; compare within tolerance.

**Plotting lives elsewhere.** `gonum/plot` is a separate repository and module, not part of this one. Adding it pulls a distinct dependency tree.

## When to Use / When Not

**Use when:**
- You need real linear algebra, stats, or optimization in a Go service without shelling out to Python.
- You want a dependency-free, cross-compilable numeric stack (pure-Go backend, no cgo).
- Your problem is `float64` dense/sparse-ish matrices, distributions, graph algorithms, or gradient/derivative-free optimization.
- You value staying inside one language and toolchain for deployment simplicity.

**Avoid when:**
- You need a mature autodiff / deep-learning tensor stack — gonum is not that.
- You need `float32`/GPU/large-tensor throughput; the `float64`, CPU, no-generics design fights you.
- Python's NumPy/SciPy/PyTorch already solve your problem and Go is not a hard requirement — the ecosystem there is far deeper.
- You require a 1.0 API-stability contract for long-lived code you will not revisit.

## Alternatives

- gorgonia/gorgonia — use when you need computation graphs, automatic differentiation, and ML/tensor ops rather than classical numerics.
- gonum/plot — use alongside gonum when you need to render charts; it is the project's separate plotting module.
- go-gota/gota — use when you want dataframe/series ergonomics (CSV, filtering, grouping) closer to pandas than to raw matrices.
- gonum/netlib — not an alternative but a backend: use when the pure-Go BLAS is too slow and you accept cgo + a native BLAS.
- NumPy/SciPy (Python) — use when Go is not a requirement and you want the deepest, best-documented numeric ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| monorepo | 2017-03 | `gonum/gonum` created; earlier separate numeric repos consolidated into one module[^1]. |
| v0.6.0 | 2019-11 | Go-modules-era tagged release under `gonum.org/v1/gonum`. |
| v0.8.0 | 2020-08 | Six-month, Go-aligned cadence (Feb/Aug) established[^2]. |
| v0.12.0 | 2022-11 | Continued matrix/stat/graph refinement in the v0 line. |
| v0.14.0 | 2023-11 | Ongoing minor release. |
| v0.15.0 | 2024-05 | Ongoing minor release. |

## References

[^1]: Gonum project overview and package index. https://www.gonum.org/
[^2]: Gonum README — installation, supported Go versions, six-month release schedule, build tags, floating-point/stability caveats. https://github.com/gonum/gonum/blob/master/README.md

## Tags

go, golang, numerical-computing, linear-algebra, matrices, statistics, optimization, graph-theory, scientific-computing, blas-lapack, data-analysis
