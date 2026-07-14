# scipy/scipy

> The standard library of scientific and numerical algorithms for Python, built on top of NumPy arrays.

[GitHub repo](https://github.com/scipy/scipy) ·
[Official website](https://scipy.org) ·
[License: BSD-3-Clause](https://github.com/scipy/scipy/blob/main/LICENSE.txt)

## Overview

SciPy is a collection of numerical algorithms layered on the NumPy array
type: optimization, integration, interpolation, linear algebra, signal and
image processing, sparse matrices, spatial data structures, statistics, and
special functions. It began in 2001 as a way to bundle the scattered
scientific Python extension modules of the era into one package, and has since
become the reference implementation most of the domain-specific ecosystem
(scikit-learn, statsmodels, scikit-image, and many others) builds on[^1]. The
SciPy 1.0 milestone landed in 2017, sixteen years after the first release, and
the project documented its architecture and history in a 2020 *Nature Methods*
paper[^2].

The defining characteristic is that SciPy is mostly a thin, well-tested Python
layer over compiled code. A large fraction of the library is wrappers around
battle-tested Fortran and C: LAPACK and BLAS for linear algebra, QUADPACK for
integration, MINPACK and later trust-region codes for optimization, FFTPACK
(now `pocketfft`) for transforms, ARPACK for sparse eigenproblems. This is its
strength (decades of numerical robustness you do not have to reimplement) and
its friction (building from source needs a Fortran and C toolchain, and the
numerical behavior you get depends on which BLAS backend is linked).

SciPy is deliberately conservative. It is a NumFOCUS-sponsored, community-run
project with a strong backward-compatibility culture; APIs change slowly and
deprecations run for several releases. It is not where you go for GPUs,
autodiff, or the newest ML method — it is the stable substrate underneath
those tools.

## Getting Started

```bash
pip install scipy          # pulls a wheel with OpenBLAS bundled
# or
conda install scipy        # via conda-forge, links the channel's BLAS
```

```python
import numpy as np
from scipy import optimize, integrate, stats

# Curve fitting: least-squares fit of a model to noisy data
def model(x, a, b):
    return a * np.exp(-b * x)

x = np.linspace(0, 4, 50)
y = model(x, 2.5, 1.3) + 0.05 * np.random.randn(x.size)
params, cov = optimize.curve_fit(model, x, y, p0=[1.0, 1.0])

# Definite integral of the fitted model
area, err = integrate.quad(lambda t: model(t, *params), 0, 4)

# A two-sample t-test
t, p = stats.ttest_ind(y[:25], y[25:])
```

## Architecture / How It Works

SciPy is organized as a set of loosely coupled subpackages under one
namespace. Each is imported explicitly (`from scipy import stats`); importing
`scipy` alone gives you almost nothing. The major subpackages are `cluster`,
`constants`, `fft`, `integrate`, `interpolate`, `io`, `linalg`, `ndimage`,
`optimize`, `signal`, `sparse`, `spatial`, `special`, and `stats`.

The dependency floor is NumPy: SciPy consumes and returns `ndarray`s and shares
its dtype and broadcasting model. Beneath the Python layer sit three kinds of
compiled code — vendored Fortran/C numerical libraries, Cython extension
modules, and (increasingly) C++ — glued to Python via `f2py`, Cython, and
`pybind11`. `scipy.linalg` is essentially a typed, higher-level interface to
LAPACK/BLAS that exposes routines NumPy's `linalg` does not.

The build system is the part most contributors and packagers actually feel.
SciPy moved off the old `distutils`/`numpy.distutils` machinery to **Meson**
(with `meson-python` as the build backend) in the 1.9 series[^3], because
`distutils` was removed from Python's standard library and never handled
Fortran and mixed-language builds well. A from-source build therefore needs
Meson, Ninja, a C/C++ compiler, and a Fortran compiler (gfortran) plus a BLAS
and LAPACK to link against.

`scipy.sparse` is its own world: multiple matrix storage formats (CSR, CSC,
COO, LIL, DOK, and others), each with different performance tradeoffs, plus a
newer `sparse.linalg` and the array-style sparse API that is gradually
superseding the older `spmatrix` classes. Choosing the wrong format for your
access pattern is one of the most common performance mistakes in SciPy code.

## Production Notes

**Your BLAS backend is a real dependency.** Numerical results, thread counts,
and speed all depend on whether SciPy is linked against OpenBLAS (the pip
wheels), Intel MKL (some conda builds), Apple Accelerate, or a system BLAS.
Different backends can give bit-level-different results and very different
performance on the same code. This matters for reproducibility across machines.

**Thread oversubscription.** BLAS libraries spawn their own threads. Running
SciPy calls inside a `multiprocessing` or `joblib` parallel loop can produce
N×M threads and thrash the CPU. `threadpoolctl` is the standard tool for
capping BLAS threads; scikit-learn's docs describe the same footgun.

**Building from source is the hard path.** Because of the Fortran + BLAS
requirement, `pip install` from an sdist on a platform without a wheel will try
to compile the whole numerical stack. Prefer wheels (PyPI) or conda-forge. If
you must build, install a Fortran compiler and a dev BLAS/LAPACK first; the
Meson error messages when these are missing are not always obvious.

**Backward compatibility is strong but not absolute.** Deprecations run for
multiple releases with warnings, but SciPy does remove things: `scipy.misc`
was deprecated and removed, various `stats` functions have been renamed or had
their defaults changed, and the sparse matrix classes are on a long path toward
the sparse *array* API. Pin your SciPy version and read the release notes on
minor-version bumps — the changelog is detailed for exactly this reason.

**Not a GPU or autodiff library.** SciPy runs on CPU with NumPy arrays. There
is ongoing work on the Python **array API standard** so parts of SciPy can
accept alternative array backends, but broad GPU support is not there — reach
for CuPy, JAX, or PyTorch when you need accelerators or gradients.

**`scipy.stats` is large and uneven.** It is one of the most-used subpackages
and one where API conventions vary by function (some return named tuples,
distributions use the `rv_continuous`/`rv_discrete` framework, argument
conventions differ). Read each function's docstring rather than assuming.

## When to Use / When Not

**Use when:**
- You need vetted implementations of classic numerical algorithms
  (optimization, integration, interpolation, linear algebra, FFTs, signal
  processing) on CPU with NumPy arrays.
- You want statistical tests, distributions, and sparse-matrix support without
  pulling in a heavier framework.
- You are building a domain library and want a stable, conservatively
  maintained foundation to depend on.

**Avoid when:**
- You need GPU acceleration or automatic differentiation — use JAX, PyTorch,
  or CuPy.
- Your problem is machine learning modeling rather than raw numerics — use
  scikit-learn, statsmodels, or a deep-learning framework.
- You only need array manipulation with no algorithms — plain NumPy is enough
  and lighter.
- You need symbolic math — use SymPy.

## Alternatives

- numpy/numpy — the array foundation SciPy sits on; use it directly when you
  only need array ops and not the algorithm layer.
- scikit-learn/scikit-learn — use instead when the task is ML modeling
  (classifiers, clustering pipelines, model selection) rather than raw numerics.
- statsmodels/statsmodels — use instead when you need statistical models,
  econometrics, and inference-grade summaries beyond `scipy.stats` tests.
- sympy/sympy — use instead when you need symbolic/exact math rather than
  floating-point numerics.
- google/jax — use instead when you need the same numerical style plus GPU/TPU
  and autodiff (`jax.scipy` reimplements a subset of the API).
- cupy/cupy — use instead when you want NumPy/SciPy-compatible APIs running on
  NVIDIA GPUs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2001 | First release; bundles early scientific Python modules[^1]. |
| 0.8 | 2010 | Broad subpackage set stabilizes on NumPy. |
| 1.0 | 2017-10 | First 1.0 after 16 years; API-stability commitment[^2]. |
| 1.3 | 2019-05 | Python 2 support dropped. |
| 1.6 | 2020-12 | Continued modernization of `stats` and `optimize`. |
| 1.9 | 2022-07 | Build system switched to Meson / meson-python[^3]. |
| 1.11 | 2023-06 | Sparse array API and array-API-standard work advance. |

## References

[^1]: SciPy documentation, "About SciPy" and project history. https://scipy.org/about/
[^2]: Pauli Virtanen et al., "SciPy 1.0: fundamental algorithms for scientific computing in Python," *Nature Methods* 17, 261–272 (2020). https://www.nature.com/articles/s41592-019-0686-2
[^3]: SciPy release notes, "SciPy 1.9.0" — Meson build system. https://docs.scipy.org/doc/scipy/release/1.9.0-notes.html

## Tags

python, scientific-computing, numerical-methods, linear-algebra, optimization, statistics, signal-processing, numpy, fortran, bsd-licensed, sparse-matrices
