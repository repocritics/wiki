# PyO3/maturin

> Build and publish Python wheels from Rust crates (pyo3, cffi, uniffi) with a zero-config PEP 517 backend.

[GitHub repo](https://github.com/PyO3/maturin) ·
[Official website](https://maturin.rs) ·
[License: Apache-2.0 OR MIT](https://github.com/PyO3/maturin/blob/main/license-apache)

## Overview

maturin builds Rust code into installable Python packages. Given a Cargo
project with pyo3, cffi, or uniffi bindings (or a plain Rust binary), it
compiles the crate, packages the resulting native extension into a wheel with
the correct platform tags, and can upload it to PyPI. It doubles as a PEP 517
build backend, so `pip install .` on a Rust-backed project just works without a
`setup.py`. It was originally named `pyo3-pack` and renamed to maturin during
the 0.x series[^1].

The defining design choice is **minimal configuration**. maturin auto-detects
the binding type from the crate's dependencies, infers package and module names
from `Cargo.toml`, and merges Python metadata from `pyproject.toml` (PEP 621)
on top of Cargo metadata. For the common case — a single pyo3 extension module
— there is almost nothing to configure. That simplicity is also the tradeoff:
projects that need fine-grained control over the wheel (custom data files,
unusual layouts, non-standard ABI handling) push against the tool's opinions,
and the escape hatches live in a `[tool.maturin]` table that grows steadily.

maturin targets Python 3.8+ on Windows, Linux, macOS, and FreeBSD, with basic
PyPy and GraalPy support[^2]. It is the de facto standard build tool in the
PyO3 ecosystem and underpins a large share of Rust-accelerated Python packages
(polars, pydantic-core, ruff, cryptography-adjacent crates, and many others).

## Getting Started

```shell
# recommended: isolated install via pipx or uv
pipx install maturin
uv tool install maturin
# or plain: pip install maturin
```

Scaffold and iterate on a pyo3 project:

```shell
maturin new -b pyo3 my_project
cd my_project
maturin develop      # compile + install into the active virtualenv
python -c "import my_project; print(my_project.sum_as_string(1, 2))"
```

The generated `src/lib.rs`:

```rust
use pyo3::prelude::*;

#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

#[pymodule]
fn my_project(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    Ok(())
}
```

The `pyproject.toml` that makes `pip install .` work:

```toml
[build-system]
requires = ["maturin>=1.0,<2.0"]
build-backend = "maturin"

[project]
name = "my_project"
requires-python = ">=3.8"
```

## Architecture / How It Works

maturin is both a **CLI** and a **PEP 517 build backend**; the backend path
(`build_wheel` / `build_sdist`) reuses the same core as the `maturin build`
command. The pipeline is:

1. **Detect bindings** — pyo3/PyO3 abi3, cffi, uniffi, or `bin`. pyo3 and
   uniffi are auto-detected from dependencies; cffi and `bin` must be declared
   via `bindings` because they cannot be inferred[^2].
2. **Compile** — invoke `cargo build` (release when `-r`/`--release` is passed;
   sdist-triggered builds are always release[^3]). The output is a `cdylib`
   (`.so`/`.pyd`/`.dylib`) or an executable.
3. **Tag & repair** — maturin contains a **reimplementation of auditwheel**.
   It inspects the extension's dynamic dependencies and assigns the strongest
   valid platform tag (a `manylinux_*` tag if compliant, else the generic
   `linux` tag). This avoids needing the separate `auditwheel` tool for the
   common case[^4].
4. **Package** — assemble the wheel (native module plus any pure-Python sources
   from a mixed layout) or a source distribution equivalent to `cargo package`.

Three commands cover the workflow: `maturin build` (produces wheels in
`target/wheels`, publishes nothing), `maturin develop` (compiles and installs
straight into the current virtualenv — the fast inner loop), and
`maturin publish` (upload; the docs now steer users toward `uv publish`
instead).

**abi3 / stable ABI.** With pyo3's `abi3` feature (e.g. `abi3-py38`), maturin
builds a single wheel usable across all Python versions ≥ the floor, using
CPython's stable ABI. Without it, you get one wheel per interpreter version.

**Mixed Rust/Python projects.** A Python package directory alongside
`Cargo.toml` (or pointed to by `tool.maturin.python-source`) is merged into the
wheel, with the native module placed inside it. Getting this layout wrong is a
well-known `ImportError` footgun that the `python-source` split avoids[^5].

**Cross-compilation.** maturin supports building portable Linux wheels with
`--zig` (using Zig as the linker/sysroot to satisfy manylinux without a
container) and Windows cross-builds via `cargo-xwin`. The
[PyO3/maturin-action](https://github.com/PyO3/maturin-action) GitHub Action
wraps the whole matrix for CI.

## Production Notes

**manylinux is the hard part.** Native Linux wheels on PyPI must link only a
small set of ubiquitous libraries. Since Rust 1.64 the compiler requires glibc
≥ 2.17, so **manylinux2014 is the practical floor**[^6]. To ship broadly
usable Linux wheels you must either build inside a manylinux container (the
`ghcr.io/pyo3/maturin` image is manylinux2014-based) or build with `--zig`.
Build against your host's newer glibc and maturin will (correctly) demote the
wheel to the `linux` tag, which PyPI will reject on upload.

**`develop` is not `build`.** `maturin develop` is faster but does not exercise
the full `pip install` path — it skips some packaging behavior. Validate
release artifacts by installing an actual built wheel before publishing, not by
relying on the develop install.

**abi3 has real costs.** The single-wheel convenience means you are restricted
to the stable ABI subset: APIs outside abi3 are unavailable, and you forgo
per-version optimizations. For performance-critical extensions, version-specific
wheels can be meaningfully faster; abi3 is a distribution-simplicity tradeoff,
not a free win.

**Pin the backend.** Use `requires = ["maturin>=1.0,<2.0"]` in
`[build-system]`. maturin follows semver at the 1.x line, but the build backend
is load-bearing for reproducible installs, and an unbounded range invites a
future 2.0 to silently change build semantics.

**Interpreter coverage is uneven.** CPython is first-class; PyPy and GraalPy
are "basic support." Cross-interpreter matrices frequently surface gaps, and
free-threaded (no-GIL) CPython support tracks pyo3's own maturity rather than
maturin's.

**CI is where this lives.** Local `maturin build` is fine for one platform, but
real distribution means a per-OS, per-arch, per-interpreter matrix. Most
projects delegate that to `maturin-action` or `cibuildwheel`; hand-rolling it
is a common source of missing or mistagged wheels.

## When to Use / When Not

**Use when:**
- You are writing a Python extension in Rust with pyo3, cffi, or uniffi.
- You want `pip install .` / a PEP 517 backend with near-zero config.
- You need to publish wheels across CPython versions, ideally as one abi3 wheel.
- You want manylinux tagging without wiring up auditwheel yourself.

**Avoid when:**
- Your native code is C/C++/CMake, not Rust — use scikit-build-core.
- You are committed to a setuptools build and only want to bolt on a Rust
  module — setuptools-rust fits that flow better.
- You need packaging control beyond `[tool.maturin]`'s knobs (exotic layouts,
  bespoke ABI handling) — you may end up fighting the tool's defaults.

## Alternatives

- PyO3/setuptools-rust — use when you already have a setuptools/`setup.py`
  build and want to add a Rust extension without switching backends.
- pypa/cibuildwheel — use to drive the multi-OS/arch/interpreter wheel matrix
  in CI; complementary to maturin rather than a replacement.
- pypa/auditwheel — use directly when you need fine control over Linux library
  bundling and repair beyond maturin's built-in reimplementation.
- scikit-build-core — use when the compiled code is C/C++/Fortran via CMake
  instead of Rust.
- astral-sh/uv — use for the publish/build-frontend side; the maturin docs now
  recommend `uv publish` over `maturin publish`.

## History

| Version | Date | Notes |
|---------|------|-------|
| pyo3-pack | 2018-07 | Initial project under the pyo3-pack name[^1]. |
| (rename) | ~2019–2020 | Renamed pyo3-pack → maturin during the 0.x series. |
| 0.x | 2019–2023 | Bindings auto-detection, manylinux/auditwheel reimpl, PEP 621, Zig cross-compile matured across the 0.x line. |
| 1.0 | 2023-04 | First stable major; PEP 517 backend and CLI declared stable[^7]. |
| 1.x | 2023–2026 | uniffi bindings, GraalPy support, ongoing abi3 and free-threaded work. |

## References

[^1]: maturin README — "_formerly pyo3-pack_". https://github.com/PyO3/maturin
[^2]: maturin README — supported platforms, Python 3.8+, PyPy/GraalPy, and bindings detection. https://github.com/PyO3/maturin
[^3]: maturin docs — source distribution builds are always in release mode. https://maturin.rs/
[^4]: maturin README — "maturin contains a reimplementation of auditwheel [that] automatically checks the generated library and gives the wheel the proper platform tag." https://github.com/PyO3/maturin
[^5]: PyO3/maturin issue #490 — mixed-layout ImportError pitfall, mitigated by `python-source`. https://github.com/PyO3/maturin/issues/490
[^6]: Rust blog — "Increasing the minimum supported glibc" (Rust 1.64 requires glibc ≥ 2.17). https://blog.rust-lang.org/2022/08/01/Increasing-glibc-kernel-requirements.html
[^7]: maturin User Guide and changelog. https://maturin.rs/

## Tags

rust, python, packaging, pyo3, wheels, build-backend, ffi, manylinux, cross-compilation, pep-517, native-extensions
