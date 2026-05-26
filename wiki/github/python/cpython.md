# python/cpython

> The reference implementation of the Python programming language.

[GitHub repo](https://github.com/python/cpython) ·
[Official website](https://www.python.org) ·
[License: PSF-2.0](https://github.com/python/cpython/blob/main/LICENSE)

## Overview

CPython is the reference implementation of Python — the interpreter, standard library, and build system written in C and Python itself, maintained by the Python core developers under the Python Software Foundation[^1]. CPython is what people mean when they say "Python" in production: PyPy, Jython, IronPython, and RustPython exist but combined hold a low single-digit market share.

Three workstreams define the modern CPython era:

1. **Faster CPython** (Mark Shannon, Guido van Rossum at Microsoft 2021–2024) — incremental interpreter speedups via specialization, inline caches, and adaptive bytecode. The headline target was 5× over Python 3.10; 3.13 has delivered roughly 1.5–2× depending on workload[^2].
2. **PEP 703 free-threaded Python** — removing the GIL. Officially experimental in 3.13 as a separate build (`python3.13t`); on track to become a supported (still optional) build in 3.14[^3].
3. **PEP 744 JIT** — a copy-and-patch JIT compiler. Experimental in 3.13, expected to mature through 3.14/3.15. Performance gains have been modest in 3.13 but the infrastructure is in place.

Together these represent the largest implementation-level change to CPython since the 2 → 3 transition.

## Getting Started

Use `uv` (Rust-based, fastest) or `pyenv` for version management:

```bash
# uv (recommended in 2025+)
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.13
uv venv
source .venv/bin/activate
```

A minimal Python program:

```python
# greet.py
from dataclasses import dataclass

@dataclass
class User:
    name: str
    email: str | None = None

def greet(user: User) -> str:
    return f"Hello, {user.name}"

if __name__ == "__main__":
    print(greet(User(name="Tom")))
```

```bash
python greet.py
```

Modern dependency management:

```bash
uv init my-app
cd my-app
uv add requests pydantic
uv run main.py
```

## Architecture / How It Works

CPython is a stack-based bytecode interpreter[^4]. Source `.py` files are compiled to bytecode `.pyc` (cached in `__pycache__/`), then executed by the evaluation loop in `Python/ceval.c`. Objects are reference-counted with a cycle collector for cyclic garbage.

**The GIL (Global Interpreter Lock)** is a mutex that serializes bytecode execution across threads in a process. It exists because CPython's reference counting is not thread-safe; making it thread-safe is exactly what PEP 703 is doing[^3]. The cost of the GIL is that multi-threaded CPython does not scale CPU-bound workloads across cores; the workaround has historically been multiprocessing (separate processes with their own GILs) or moving CPU work into C extensions that release the GIL.

**Specialization (PEP 659)** — bytecode instructions adapt to runtime types. The interpreter starts with generic ops (`BINARY_OP`) and after a few executions rewrites them to type-specialized versions (`BINARY_OP_ADD_INT`). This is the core of "Faster CPython"'s wins[^2].

**The copy-and-patch JIT (PEP 744)** — at startup, the interpreter has templates for specialized hot loops. When a region becomes hot enough, the JIT patches together those templates into a contiguous machine code blob and jumps to it. Lower implementation complexity than a traditional tracing JIT (PyPy) at the cost of leaving optimizations on the table.

**Per-interpreter GIL (PEP 684)** — multiple subinterpreters in one process can each have their own GIL. Combined with PEP 734 (multiple interpreters in the stdlib), this gives a path to in-process parallelism without removing the global GIL entirely. Useful for embedding, less useful for application code.

**Packaging** has long been Python's weakest area. The PSF-blessed path in 2025 is: `pyproject.toml` for metadata, `uv` (or `poetry`, `pdm`, `hatch`) for resolution, `wheel` for distribution, `PyPI` as the registry. `setup.py` is legacy but still works.

## Production Notes

**Migrating to 3.13.** The free-threaded build is opt-in (separate binary, separate wheels with the `t` suffix). Most pure-Python code "just works"; C extensions need explicit support — NumPy, PyTorch, and similar have published roadmaps[^5]. Until the ecosystem catches up, use the GIL build (`python3.13`) for production.

**JIT in 3.13.** Off by default. Enable with `--enable-experimental-jit` at build time. Benchmarks are mixed — some workloads see 5–10% gains, others see regressions. Not yet production-ready.

**Wheel ABI tags.** With free-threading, the wheel tag matrix doubled: `cp313-cp313-linux_x86_64` (GIL build) vs `cp313t-cp313t-linux_x86_64` (free-threaded). Package authors must publish both for ecosystem support. Use `cibuildwheel` to automate.

**Memory profile.** Python's per-object overhead is substantial (~50 bytes for an int, ~64 for a small dict). Use `__slots__` for high-volume classes, `array.array` or NumPy for homogeneous numeric data, `dataclass(slots=True)` (3.10+) for combined ergonomics + memory.

**asyncio gotchas.**
- Mixing sync and async code is awkward. `asyncio.to_thread` is the escape hatch for blocking calls.
- Exception groups (PEP 654) and `TaskGroup` (3.11+) are the modern structured-concurrency primitives — prefer them over raw `asyncio.gather`.
- `asyncio.run` should be the single entry point per program; nested event loops are an anti-pattern.

**Type hints.** Slow ecosystem adoption (Python is 35 years old; many libraries predate PEP 484). `mypy --strict`, `pyright`, and `pyrefly` are the type checkers. `pyright` is generally fastest and most strict by default.

**Packaging tools matrix.** `uv` is the 2025 default for new projects — Rust-based, dramatically faster than `pip`. `poetry` and `pdm` remain widely used. Avoid mixing tools in one project; the lockfile formats are not interchangeable.

## When to Use / When Not

**Use when:**
- Data science, ML, scientific computing — the ecosystem (NumPy, PyTorch, JAX, pandas, scikit-learn) is unmatched.
- Scripting, automation, CLI tools — `stdlib` covers most needs, fast prototyping.
- Backend services where developer velocity matters more than raw throughput.
- Anywhere C extensions provide the heavy lifting (NumPy, native crypto, native parsing).

**Avoid when:**
- Cold start matters and packaging is large (serverless with 200 MB deps takes seconds to import).
- You need true multi-threaded CPU-bound scaling without rewriting in C — until the free-threaded ecosystem matures.
- Memory budget is constrained (embedded, mobile) — Python's per-object overhead is substantial.

## Alternatives

- **PyPy** — tracing JIT, dramatically faster on pure-Python loops, but lags C extension compatibility.
- **Pyston** — fork of CPython with perf patches; less active in 2025.
- **RustPython** — Rust reimplementation, useful for WASM targets, not production-ready as a CPython replacement.
- [rust-lang/rust](../rust-lang/rust.md), [golang/go](../golang/go.md) — for the systems-language alternatives.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9 | 1991-02 | First public release. |
| 2.0 | 2000-10 | List comprehensions, garbage collection. |
| 3.0 | 2008-12 | Backwards-incompatible cleanup. |
| 3.6 | 2016-12 | f-strings, ordered dicts (CPython impl detail). |
| 3.8 | 2019-10 | Walrus operator, positional-only params. |
| 3.10 | 2021-10 | Structural pattern matching (PEP 634). |
| 3.11 | 2022-10 | Faster CPython phase 1, exception groups, `TaskGroup`[^2]. |
| 3.12 | 2023-10 | Per-interpreter GIL (PEP 684), buffer protocol improvements. |
| 3.13 | 2024-10 | Free-threaded build (PEP 703), copy-and-patch JIT (PEP 744)[^3]. |
| 3.14 | 2025-10 | Free-threading toward "supported", more JIT, t-strings (PEP 750). |

## References

[^1]: Python Software Foundation, "About Python". https://www.python.org/about/
[^2]: Mark Shannon, "Faster CPython" — Python Language Summit 2021+. https://github.com/faster-cpython
[^3]: Sam Gross, "PEP 703 — Making the Global Interpreter Lock Optional". https://peps.python.org/pep-0703/
[^4]: CPython developer guide, "CPython Internals". https://devguide.python.org/internals/
[^5]: Python free-threading compatibility tracker. https://py-free-threading.github.io/
