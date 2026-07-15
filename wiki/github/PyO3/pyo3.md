# PyO3/pyo3

> Rust bindings for the CPython interpreter — write native Python extension modules in Rust, or embed Python inside a Rust binary.

[GitHub repo](https://github.com/PyO3/pyo3) ·
[Official website](https://pyo3.rs) ·
[License: Apache-2.0 OR MIT](https://github.com/PyO3/pyo3/blob/main/LICENSE-APACHE)

## Overview

PyO3 is the de facto way to bridge Rust and Python. It began in 2017 as a fork of Daniel Grunwald's `rust-cpython`, was carried by Nikolay Kim and Konstantin Schütze, and is now maintained primarily by David Hewitt[^1]. It powers the Rust cores of several widely deployed Python packages — pydantic-core, cryptography, polars, tokenizers, orjson, and tiktoken among them[^2] — which makes it one of the most load-bearing FFI layers in the Python ecosystem even though most Python developers never invoke it directly.

The library does two related jobs. The common one is building a native extension module: you annotate Rust functions and structs with macros (`#[pyfunction]`, `#[pyclass]`, `#[pymodule]`) and PyO3 generates the CPython C-API glue so Python can `import` the compiled `cdylib`. The less common one is embedding — starting a CPython interpreter from a Rust binary and calling into it. Both sit directly on the CPython C-API, so PyO3 inherits that API's constraints: reference counting, the interpreter lock, and per-version ABI breakage.

PyO3's defining tension is its pre-1.0 status against its production ubiquity. It has never shipped a 1.0 release; every `0.x` minor version carries breaking changes, and the migration guide is a first-class document rather than an afterthought[^3]. The most significant of these was the 2024 shift from the old "GIL Refs" borrowed-reference API (`&PyAny`, `&PyDict`) to the `Bound<'py, T>` smart-pointer API — a change that touched nearly every downstream project. Depending on PyO3 means accepting a steady upgrade tax in exchange for the safest available Rust/Python boundary.

## Getting Started

Requires Rust 1.83+ and CPython 3.9+ (PyPy 7.3+, GraalPy 25.0+ also supported). The path of least resistance is [maturin](https://github.com/PyO3/maturin):

```bash
mkdir string_sum && cd "$_"
python -m venv .env && source .env/bin/activate
pip install maturin
maturin init --bindings pyo3
maturin develop   # compiles the Rust cdylib and installs it into the venv
```

```rust
// src/lib.rs — a native module importable as `import string_sum`
use pyo3::prelude::*;

#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

#[pymodule]
fn string_sum(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)
}
```

```python
>>> import string_sum
>>> string_sum.sum_as_string(5, 20)
'25'
```

To embed Python instead, add `pyo3 = { version = "0.29", features = ["auto-initialize"] }` and drive the interpreter through `Python::attach(|py| { ... })`.

## Architecture / How It Works

PyO3 is a set of proc-macros plus a hand-written C-API wrapper. The macros expand into `extern "C"` trampolines matching CPython's `PyMethodDef` / `PyType_Spec` shapes; the runtime crate wraps `python3.h` in safe(r) Rust types.

**The interpreter-attachment token.** Every operation that touches Python objects requires proof that the current thread is attached to the interpreter. PyO3 encodes this as a zero-sized token, `Python<'py>`, and a lifetime `'py` that flows through every bound reference. You obtain it via `Python::attach` (historically `Python::with_gil`) and cannot manufacture one — this is how the type system prevents C-API calls from a detached thread.

**Two pointer types.** `Bound<'py, T>` is a reference-counted handle whose `'py` lifetime ties it to interpreter attachment; it is the working currency of modern PyO3. `Py<T>` is the attachment-independent form you store in Rust structs or move across threads, later re-bound with `.bind(py)`. The older "GIL Refs" API exposed plain borrows like `&PyAny` whose lifetime was tied to a `GILPool`; it was deprecated with the `Bound` migration in 0.21 and subsequently removed[^3].

**Conversions.** `IntoPyObject` / `FromPyObject` (and the `#[derive(FromPyObject)]` macro) marshal between Rust and Python types. `#[pyclass]` exposes a Rust struct as a Python type; `#[pymethods]` attaches its methods, with `#[new]`, `#[getter]`, and `#[staticmethod]` covering the dunder surface.

**The GIL and free-threading.** On standard CPython, attachment maps onto holding the Global Interpreter Lock, so `Bound` values are effectively `!Send`. PyO3 also supports free-threaded CPython (the PEP 703 `3.13t` build) where there is no GIL; the "attach/detach" naming was adopted precisely because "acquire the GIL" no longer describes what happens. Extensions must opt into free-threaded compatibility explicitly.

**ABI.** By default an extension is compiled against one specific CPython version's ABI. The `abi3` feature targets CPython's stable Limited API instead, producing a single wheel that loads on that minimum version and all later ones — at the cost of a narrower API surface.

## Production Notes

**The `extension-module` feature is not optional trivia.** When building an extension module you must enable `features = ["extension-module"]` so PyO3 does not link `libpython` (the host interpreter provides those symbols at import time). But that same setting breaks `cargo test` and `cargo run`, which have no interpreter to borrow symbols from, producing "symbol not found" / "undefined reference to `_PyExc_*`" link errors. The standard fix is to gate the feature behind an optional flag or use maturin's test path[^4]; nearly every PyO3 project hits this once.

**Upgrade tax is the dominant operational cost.** Minor versions break. The GIL-Refs → `Bound` migration (0.21+) was the largest, mechanically rewriting signatures across a codebase; teams with transitive PyO3 dependencies frequently got stuck waiting for every leaf crate to converge on the same major, since two incompatible PyO3 versions cannot coexist in one extension. Budget for it and pin deliberately.

**Attachment overhead and blocking.** On GIL-based CPython, every attached section serializes against all other Python threads. Long CPU-bound Rust work should `Python::detach` (release attachment) so other Python threads run; forgetting to do so turns a "fast Rust core" into a global stall. Conversely, free-threaded builds remove that lock but expose you to real data races in any `unsafe` or global state.

**Build and distribution.** Shipping wheels for the matrix of {CPython versions} × {OS} × {arch} is the real work; maturin plus `cibuildwheel`/GitHub Actions is the usual answer. `abi3` collapses the Python-version axis at the price of API breadth. Cross-compilation and manylinux compliance are recurring pain points.

**Panic and error semantics.** A Rust `panic!` across the FFI boundary is undefined behavior in the C-API contract; PyO3 catches panics at the boundary and converts them to Python exceptions, but you should still return `PyResult` rather than panic. `PyErr` models Python exceptions; `?` works across the boundary once conversions are in place.

## When to Use / When Not

**Use when:**
- You have a CPU-bound Python hot path (parsing, serialization, numerics) worth rewriting in Rust.
- You want to expose an existing Rust library to Python with memory safety at the boundary.
- You need a maintained, actively developed binding layer — PyO3 is where the ecosystem and the free-threaded/3.13 work land first.

**Avoid when:**
- Your bottleneck is I/O or orchestration, not CPU — the FFI and build complexity buy nothing.
- You cannot absorb periodic breaking upgrades or a Rust toolchain in CI.
- You want to bind C++ rather than Rust (use pybind11), or you want to avoid a compiled toolchain entirely (use Cython or cffi).

## Alternatives

- dgrunwald/rust-cpython — the predecessor PyO3 was forked from; effectively dormant, use PyO3 instead unless maintaining legacy code.
- pybind11/pybind11 — the same job for C++ instead of Rust; choose it when your native code is already C++.
- cython/cython — compile a Python superset to a C extension; better when you want incremental speedups without a second language toolchain.
- RustPython/RustPython — a pure-Rust Python interpreter; use it when you want to embed Python-like execution without linking CPython.
- PyO3/maturin — not a competitor but the standard build/publish companion; reach for it before hand-rolling `setuptools-rust`.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017 | Initial release; fork of `rust-cpython`[^1]. |
| 0.21 | 2024 | `Bound<'py, T>` smart-pointer API introduced; GIL-Refs deprecated[^3]. |
| 0.23 | 2024 | Free-threaded CPython (PEP 703 / 3.13t) support[^5]. |
| 0.29 | 2026 | Current line; MSRV Rust 1.83, CPython 3.9+ / PyPy 7.3 / GraalPy 25[^2]. |

Version dates are approximate to the year; see the changelog for exact release dates[^3].

## References

[^1]: PyO3 project history and maintainers. https://github.com/PyO3/pyo3/graphs/contributors
[^2]: PyO3 README — supported interpreters and downstream users. https://github.com/PyO3/pyo3/blob/main/README.md
[^3]: PyO3 migration guide and changelog. https://pyo3.rs/latest/migration.html
[^4]: PyO3 FAQ — "I can't run cargo test / linker issues." https://pyo3.rs/latest/faq.html
[^5]: PyO3 user guide — free-threaded Python support. https://pyo3.rs/latest/free-threading.html

## Tags

rust, python, ffi, bindings, cpython, python-c-api, extension-modules, native-extensions, interop, gil
