# wjakob/nanobind

> A C++/Python binding library that trades pybind11's breadth for smaller, faster, quicker-to-compile extension modules.

[GitHub repo](https://github.com/wjakob/nanobind) ·
[Documentation](https://nanobind.readthedocs.io/en/latest/) ·
[License: BSD-3-Clause](https://github.com/wjakob/nanobind/blob/master/LICENSE)

## Overview

nanobind is a header-heavy C++17 library that exposes C++ types to Python and
vice versa, using syntax nearly identical to pybind11. It is written by Wenzel
Jakob, who also authored pybind11[^1]; nanobind is his deliberate second attempt
at the problem, informed by years of watching pybind11 accumulate features and
compile-time cost. First released in 2022[^2], it reports roughly 4x faster
compilation, 5x smaller binaries, and up to 10x lower runtime dispatch overhead
than pybind11 on the project's own benchmarks[^3].

The defining tension is scope. pybind11 tries to bind almost any C++ construct
and support a wide matrix of old compilers and Python versions; nanobind
intentionally drops the long tail — pre-C++17 compilers, Python 2 and old 3.x,
some implicit STL conversions, and several rarely used casters — to keep the
template metaprogramming lean. The payoff is real, but it means nanobind is not
a drop-in replacement: migrating from pybind11 is usually mechanical yet almost
never zero-effort[^4].

Adoption skews toward performance-sensitive numerical projects. JAX, Apple's
MLX, IREE, LLVM's MLIR/XLA Python bindings, FEniCS/DOLFINx, and PennyLane have
all migrated from pybind11, frequently citing compile time and the ability to
target Python's Stable ABI as the deciding factors[^5].

## Getting Started

nanobind builds native extension modules and expects CMake (scikit-build-core
is the recommended packaging front end). Install the build-time dependency:

```bash
pip install nanobind
```

```cmake
# CMakeLists.txt
find_package(nanobind CONFIG REQUIRED)
nanobind_add_module(my_ext my_ext.cpp)
```

```cpp
// my_ext.cpp
#include <nanobind/nanobind.h>
namespace nb = nanobind;

int add(int a, int b) { return a + b; }

NB_MODULE(my_ext, m) {           // NB_MODULE, not PYBIND11_MODULE
    m.def("add", &add, "a"_a, "b"_a);
}
```

The `wjakob/nanobind_example` repository is the canonical scaffold for a
pip-installable project wired through scikit-build-core.

## Architecture / How It Works

nanobind's speed comes from doing less work per bound symbol at compile time.
Function definitions are lowered to a compact, largely non-templated dispatcher
rather than generating a bespoke call path per signature, so the number of
template instantiations — the dominant driver of pybind11 build times — grows
much more slowly. Bound C++ instances use a small, uniform payload layout
instead of pybind11's heavier per-type machinery.

Two capabilities anchor much of the recent adoption:

- **Stable ABI (`abi3`).** On Python 3.12+, nanobind can compile against the
  limited API so a single wheel runs on all future 3.12+ interpreters. This is
  the feature JAX cited when it stopped shipping one CUDA plugin per Python
  version[^5].
- **`nb::ndarray`.** A zero-copy tensor interchange type built on DLPack and the
  buffer protocol, so arrays move between NumPy, PyTorch, JAX, and TensorFlow
  without a copy. DOLFINx and MLX lean on this for multi-dimensional data.

nanobind also supports free-threaded (no-GIL) CPython builds, exposing explicit
locking primitives for binding code that must be safe once the interpreter GIL
is gone[^6]. Ownership between C++ and Python is handled through shared/unique
holders and a keep-alive mechanism analogous to pybind11's, but the internals
are documented more plainly, which migrating teams repeatedly note as easier to
reason about than pybind11's memory model.

## Production Notes

- **It is not source-compatible with pybind11.** Macros (`NB_MODULE`), namespace
  (`nb::`), and several casters differ. STL type conversions require explicitly
  including the relevant header (e.g. `nanobind/stl/vector.h`); forgetting one
  fails at compile time rather than silently. Budget real migration time.
- **Compiler and toolchain floor is higher.** C++17 is mandatory and older
  compilers are unsupported. If your build farm still targets ancient GCC or
  must produce pre-3.8 Python wheels, nanobind is off the table.
- **Stable ABI is opt-in and version-gated.** The single-wheel benefit only
  applies from Python 3.12 onward; on 3.8–3.11 you still build per-version.
- **Deliberately missing features.** Some pybind11 conveniences were removed on
  purpose (certain implicit conversions, embedding niceties). Check the "porting
  from pybind11" guide before assuming a feature exists[^4].
- **Dispatch is fast but stricter.** The leaner overload resolution can reject
  argument combinations pybind11 accepted through implicit coercion; make
  conversions explicit.
- **The binding library moves quickly.** nanobind reached 1.0 in 2023 and 2.0 in
  2024[^2]; changelogs are detailed but the pre-2.0 API saw churn. Pin the
  version and read release notes on upgrade.

## When to Use / When Not

**Use when:**
- You bind performance-critical numerical C++ and compile time or binary size
  hurts (large template-heavy projects benefit most).
- You want one Stable-ABI wheel per platform across Python 3.12+.
- You need zero-copy array interchange or free-threaded-Python support.
- You control a modern C++17 toolchain end to end.

**Avoid when:**
- You must support old compilers, Python 2, or pre-3.8 interpreters.
- You have a large existing pybind11 codebase and no appetite for a migration.
- Your native code is Rust, not C++ (use a Rust-native binder).
- You need a specific pybind11 feature nanobind intentionally dropped.

## Alternatives

- pybind/pybind11 — the mature predecessor; use it when you need the broad
  compiler/Python matrix or a feature nanobind removed.
- cython/cython — use when you're writing Python-like accelerated code rather
  than wrapping an existing C++ library.
- PyO3/pyo3 — use when the native side is Rust instead of C++.
- boostorg/python — legacy binder; only sensible if you're already deep in Boost.
- Raw CPython C API — use for a dependency-free extension when you accept far
  more boilerplate and manual reference counting.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2022 | Initial public release by Wenzel Jakob[^2]. |
| 1.0.0 | 2023 | First stable API; ndarray, Stable ABI groundwork. |
| 2.0.0 | 2024 | Major release; free-threading support and internals rework[^2]. |

## References

[^1]: pybind11, also authored by Wenzel Jakob. https://github.com/pybind/pybind11
[^2]: nanobind changelog. https://nanobind.readthedocs.io/en/latest/changelog.html
[^3]: nanobind benchmarks (compile time, binary size, runtime overhead vs pybind11 and Cython). https://nanobind.readthedocs.io/en/latest/benchmark.html
[^4]: nanobind docs, "Porting from pybind11". https://nanobind.readthedocs.io/en/latest/porting.html
[^5]: nanobind README testimonials — IREE, XLA/MLIR, MLX, JAX, FEniCS/DOLFINx, PennyLane. https://github.com/wjakob/nanobind
[^6]: nanobind docs, free-threading support. https://nanobind.readthedocs.io/en/latest/free_threaded.html

## Tags

cpp, python, bindings, cpp17, foreign-function-interface, native-extensions, pybind11-alternative, stable-abi, numerical-computing, ffi
