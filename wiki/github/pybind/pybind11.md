# pybind/pybind11

> Header-only C++11 library that exposes C++ types to Python and back, by inferring bindings from compile-time template introspection.

[GitHub repo](https://github.com/pybind/pybind11) ·
[Official website](https://pybind11.readthedocs.io/) ·
[License: BSD-3-Clause](https://github.com/pybind/pybind11/blob/master/LICENSE)

## Overview

pybind11 is a header-only library for writing Python bindings of C++ code (and, less commonly, embedding Python in C++). Created by Wenzel Jakob (EPFL) in 2015, it was conceived as a stripped-down Boost.Python: same idea — infer binding signatures from C++ types using templates — but depending only on the C++11 standard library and CPython, not the entire Boost suite[^1]. The core headers are on the order of a few thousand lines and produce noticeably smaller binaries and faster compiles than Boost.Python did.

The defining tradeoff is **compile-time metaprogramming**. Because bindings are expressed as ordinary C++ (`py::class_<Foo>(m, "Foo").def(...)`) and resolved through heavy template machinery, you get type safety, no code-generation step, and no external tool in your build — but you pay in compile time, template-heavy error messages, and binary bloat that scales with the number of bound types. There is no IDL and no separate generator: the binding *is* C++.

It is the default choice for "I have an existing C++ library and want a Pythonic wrapper" across scientific and systems software. With ~18,000 stars, ~2,300 forks, and commits within days of this writing, it is actively maintained[^2]. Note that the original author has since built a successor, nanobind, targeting newer compilers and lower overhead; pybind11 remains the conservative, maximally-compatible option.

## Getting Started

```bash
pip install pybind11        # ships headers + CMake/pkg-config helpers
```

```cpp
// example.cpp
#include <pybind11/pybind11.h>
namespace py = pybind11;

int add(int i, int j) { return i + j; }

struct Greeter {
    Greeter(std::string name) : name_(std::move(name)) {}
    std::string greet() const { return "Hello, " + name_; }
    std::string name_;
};

PYBIND11_MODULE(example, m) {          // module name MUST match the .so basename
    m.def("add", &add, "Add two ints", py::arg("i"), py::arg("j"));

    py::class_<Greeter>(m, "Greeter")
        .def(py::init<std::string>())
        .def("greet", &Greeter::greet)
        .def_readwrite("name", &Greeter::name_);
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(example LANGUAGES CXX)
find_package(pybind11 CONFIG REQUIRED)
pybind11_add_module(example example.cpp)
```

```python
import example
print(example.add(2, 3))          # 5
print(example.Greeter("Tom").greet())
```

## Architecture / How It Works

The whole library is templates and a small shared runtime. Nothing is generated ahead of time.

- **Type casters** (`type_caster<T>`) are the core abstraction: a specialization that knows how to convert a C++ `T` to/from a Python object. Built-ins (ints, strings, `std::vector`, `std::map`, `std::optional`, `std::function`, etc.) ship as casters in separate headers you opt into (`<pybind11/stl.h>`, `<pybind11/functional.h>`, `<pybind11/eigen.h>`). STL casters **copy** by default — a `std::vector` crossing the boundary is duplicated, not shared.
- **`py::class_<T, Holder>`** registers a custom type. The *holder* (default `std::unique_ptr<T>`, or `std::shared_ptr<T>`, or a custom smart pointer) governs ownership. Mismatched holders are a classic source of double-frees and dangling references.
- **The `internals` struct** is a process-global registry of all bound types, shared across every pybind11 extension module in the interpreter via a capsule stored in `sys`. This is what lets a type bound in module A be returned from module B. It is also the source of the ABI-compatibility rule below.
- **Function dispatch** — overloads are stored as a linked list; on call, pybind11 tries each signature in order, running the argument casters until one matches. This makes overload resolution a runtime cost, not a compile-time one.
- **`PYBIND11_MODULE`** expands to the CPython init function. Signatures and docstrings are computed with `constexpr` at compile time to keep the runtime and binary lean.
- **The GIL** is not abstracted away: you hold it whenever you touch Python objects. `py::gil_scoped_release` / `py::gil_scoped_acquire` bracket long C++ work so other threads run.

Virtual functions overridable from Python require a *trampoline* class (`PYBIND11_OVERRIDE`) that forwards C++ virtual calls back into Python — an explicit, somewhat verbose pattern.

## Production Notes

**ABI compatibility is version- and flag-sensitive.** Two extension modules that share C++ types must be built with compatible pybind11 internals versions *and* compatible compiler/stdlib ABI, otherwise they silently fail to recognize each other's types at runtime (a type looks "unregistered"). The internals version is bumped between some releases specifically to prevent mixing incompatible ABIs. In monorepos, pin one pybind11 version everywhere.

**Compile time and binary size scale with bound surface.** Every `.def` instantiates templates. Large binding files (hundreds of methods) can dominate build time and produce multi-megabyte `.so` files. Mitigations: split bindings across translation units, use `py::class_` sparingly for hot types, disable unused STL headers, and consider `-Os`. This is the most common complaint and the main reason teams evaluate nanobind.

**STL copies are easy to miss.** Returning `std::vector<double>` copies the whole vector into a Python list. For large numeric data use the buffer protocol / NumPy integration (`py::array_t`) or Eigen support to share memory instead. `py::bind_vector` gives an opaque, reference-semantic container when you genuinely want no copy.

**GIL and threading.** Callbacks from C++ threads into Python must reacquire the GIL; forgetting `gil_scoped_acquire` is undefined behavior. Long compute should release the GIL. Free-threaded (no-GIL) CPython 3.13+ support exists but is newer and less battle-tested.

**Lifetime bugs.** Returning a raw pointer/reference to a member without a `return_value_policy` (e.g. `reference_internal`) is the canonical footgun — Python may free the parent while a child reference is live, or double-free depending on the holder. Read the return-value-policy table before binding anything that returns references.

**Version/toolchain caveats.** NumPy 2 requires pybind11 2.12+[^3]. Supported runtimes are CPython 3.8+, PyPy, and GraalPy[^4]; older interpreters need older pybind11. C++11 is the floor, but enabling newer standards changes ABI, so keep it consistent across a shared type universe.

## When to Use / When Not

**Use when:**
- You have an existing C++ codebase and want idiomatic Python bindings without a separate IDL or generator.
- You need fine control over ownership, GIL, and conversions, and are comfortable in C++.
- You want zero-copy NumPy/Eigen interop for numeric data.
- You need maximum compiler/interpreter compatibility (older toolchains, PyPy, GraalPy).

**Avoid when:**
- You're on a modern compiler and care most about compile time / binary size — nanobind is smaller and faster for the same model.
- You want to bind Rust, not C++ (use PyO3).
- You want automatic wrapping of a large C API without writing C++ (SWIG or cffi/ctypes fit better).
- Your "binding" is really just calling a few C functions — ctypes/cffi avoid a C++ toolchain entirely.

## Alternatives

- wjakob/nanobind — successor by the same author; C++17, much smaller binaries and faster compiles. Use instead when you control the toolchain and don't need pybind11's legacy compatibility.
- boostorg/python — the original template-binding library. Use only if you're already deep in Boost; otherwise pybind11 supersedes it.
- PyO3/pyo3 — bindings for Rust, not C++. Use when the native code is Rust.
- swig/swig — multi-language generator driven by interface files. Use when you must target many languages or wrap a large C API without hand-writing C++.
- python-cffi/cffi — call C (not C++) from Python with no compiler-side glue. Use for thin C-ABI wrappers.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07 | Repository created; header-only Boost.Python alternative[^1]. |
| 1.0 | 2016 | First tagged public release. |
| 2.0 | 2017-01 | Major API cleanup; start of the long-lived 2.x line. |
| 2.6 | 2020-10 | Interpreter-support modernization; new caster features. |
| 2.10 | 2022-07 | Continued 2.x maintenance; compiler/runtime updates. |
| 2.12 | 2024 | NumPy 2 support[^3]. |
| 3.0 | 2025 | v3 line; modernized internals, C++11 floor retained[^4]. |

## References

[^1]: pybind11 README — origin and design goals ("a tiny self-contained version of Boost.Python"). https://github.com/pybind/pybind11#readme
[^2]: GitHub repository metadata (stars, forks, last push) as of 2026-07. https://github.com/pybind/pybind11
[^3]: pybind11 documentation and README — "Integrated NumPy support (NumPy 2 requires pybind11 2.12+)." https://pybind11.readthedocs.io/en/stable/
[^4]: pybind11 v3 README — supported runtimes (CPython 3.8+, PyPy3 7.3.17+, GraalPy 24.1+) and C++11 baseline. https://github.com/pybind/pybind11/blob/master/README.rst

## Tags

python, cpp, bindings, ffi, header-only, cpython, native-extensions, interop, scientific-computing, numpy, cmake
