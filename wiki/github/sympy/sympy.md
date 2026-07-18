# sympy/sympy

> A computer algebra system written in pure Python — symbolic math as an importable library rather than a standalone application.

[GitHub repo](https://github.com/sympy/sympy) ·
[Official website](https://sympy.org/) ·
[License: BSD-3-Clause](https://github.com/sympy/sympy/blob/master/LICENSE)

## Overview

SymPy is a computer algebra system (CAS): symbolic differentiation and integration, equation solving, series expansion, polynomial algebra, matrices, and pretty/LaTeX printing — all as exact symbolic manipulation, not floating point. Started by Ondřej Čertík in 2005, kept alive by an unusually long run of Google Summer of Code cohorts (every year since 2007), and led by Aaron Meurer since 2011, it is a NumFOCUS-sponsored project with a 2017 PeerJ paper as its canonical citation[^1][^2].

The defining tradeoff is in the tagline: *pure Python*. Where Mathematica, Maple, and Maxima are standalone applications with their own languages, SymPy is a zero-compilation `pip install` that lives inside the Python ecosystem — which is why it is the default symbolic backend for Jupyter notebooks, textbooks, and downstream libraries (SageMath embeds it; PyDy and countless physics courses build on it). The cost is speed: core operations run one to two orders of magnitude slower than C/C++-backed systems, which the project itself acknowledges by maintaining SymEngine, an optional C++ replacement core[^4].

Its 14.8k stars understate its reach — SymPy is infrastructure, pulled in as a dependency far more often than starred. Development is active (pushed today as of this writing), but the ~5,800 open issues reflect two decades of accumulated surface area across dozens of mathematical domains, many historically maintained by whoever last did a GSoC project there.

## Getting Started

```bash
pip install sympy        # pure Python, no compiler needed
pip install gmpy2        # optional speedup for integer/polynomial arithmetic
```

```python
from sympy import symbols, integrate, solve, lambdify, sin, exp, oo

x, y = symbols("x y")

expr = x**2 * sin(x)
integrate(expr, x)                    # -x**2*cos(x) + 2*x*sin(x) + 2*cos(x)
integrate(exp(-x**2), (x, -oo, oo))   # sqrt(pi) — exact, not 1.7724...

solve(x**2 - 2*x - 8, x)              # [-2, 4]

f = lambdify(x, expr, "numpy")        # compile to a NumPy-vectorized function
```

The bundled `isympy` shell preloads the namespace and enables pretty printing for interactive use.

## Architecture / How It Works

Everything is an immutable expression tree. Every object derives from `Basic`; scalars from `Expr`. A node is `(func, args)` — `(x + 1)**2` is `Pow(Add(x, 1), 2)`. Python operators are overloaded to build trees, with *automatic evaluation* in constructors: `x + x` becomes `2*x` at construction time and canonical ordering is applied; suspending this requires `evaluate=False` tricks. `sympify()` coerces Python objects into this world, which creates the ecosystem's most famous gotcha: `==` is *structural* equality, not mathematical equivalence — `(x+1)**2 == x**2 + 2*x + 1` is `False`; you must check `simplify(a - b) == 0` or use `.equals()`[^3].

Two assumption systems coexist. The "old" system attaches predicates at symbol creation (`Symbol("x", positive=True)`) and drives most of the core's simplification decisions; the "new" system (`Q`, `ask()`) was designed to replace it over a decade ago and remains only partially adopted. This is the project's longest-running unfinished migration and a real source of confusion: which assumptions a given simplification consults depends on the code path.

The `polys` module is the computational workhorse: its own tower of coefficient domains (`ZZ`, `QQ`, extension fields, polynomial rings), dense and sparse representations, and optional `gmpy2` ground types. Groebner bases, factorization, `cancel`, `factor`, and much of `solve` bottom out here. Symbolic integration combines pattern matching, a partial Risch algorithm implementation, and Meijer G-function lookup. Floating-point evaluation (`evalf`) delegates to mpmath, SymPy's arbitrary-precision sibling library (bundled until 1.0, an external dependency since)[^5]. The printing subsystem doubles as a code generator: LaTeX, Unicode pretty-printing, and C/Fortran/NumPy printers, with `lambdify` as the everyday bridge from symbolic results to fast numeric functions.

## Production Notes

**Speed is the tax you pay for pure Python.** Symbolic manipulation on large expressions is slow, and *expression swell* (intermediate results growing exponentially) compounds it. Install `gmpy2` (large speedups in `polys`), prefer targeted transformations, and for hot symbolic cores consider `symengine.py`, a mostly-compatible C++ drop-in that covers core arithmetic but not the full API[^4].

**`simplify()` is a black box.** It tries a battery of transformations and returns whatever scores best — slow and unpredictable on large expressions. Production code should use the targeted functions (`expand`, `factor`, `cancel`, `trigsimp`, `radsimp`, `powsimp`) whose behavior is specified.

**Assumptions silently gate simplification.** `sqrt(x**2)` does *not* become `x` for a default symbol (which may be complex); it does with `Symbol("x", positive=True)`. Getting assumptions right at symbol creation is the difference between results that simplify and results that stall.

**Float contamination.** `x + 0.1` embeds an inexact `Float` that poisons exact arithmetic downstream; use `Rational(1, 10)` or `S("1/10")`. Note that 1.13 stopped treating `Float` and exact numbers of equal value as `==`-equal — a deliberate correctness change that broke downstream test suites[^6].

**`solve` vs `solveset`.** `solve` is heuristic and returns differently shaped results (list, dict, list of tuples) depending on input; `solveset` returns proper set objects and handles infinite solution sets, but still does not cover everything `solve` does. Fifteen years in, you need both.

**Long-running processes.** The global expression cache (`cacheit`) grows without bound by default; symbolic-math web services should clear it periodically (`sympy.core.cache.clear_cache()`) or disable it via `SYMPY_USE_CACHE=no` at a real performance cost. `import sympy` itself takes noticeable time — keep it out of cold-start-sensitive paths.

## When to Use / When Not

**Use when:**
- You need exact symbolic results (derivatives, integrals, limits, closed-form solutions) inside a Python program or notebook.
- You want a symbolic-to-numeric pipeline: derive expressions symbolically, `lambdify` to NumPy for evaluation (common in physics, controls, ML research).
- You need LaTeX generation or code generation from mathematical expressions.
- Licensing matters: BSD-3-Clause vs. Mathematica/Maple seats.

**Avoid when:**
- Raw symbolic throughput is the bottleneck — dedicated CASs or SymEngine-class cores are 10–100× faster on heavy manipulation.
- You only need numerics: NumPy/SciPy (or mpmath alone for precision) without symbolic overhead.
- You need best-in-class symbolic integration or ODE solving on hard cases — Mathematica and specialized systems still win there.

## Alternatives

- symengine/symengine — C++ symbolic core with Python bindings; use it when SymPy's core arithmetic is your measured bottleneck and its API subset suffices.
- sagemath/sage — full open-source mathematics distribution (wrapping SymPy, Maxima, GAP, PARI); use it when you want a Mathematica-scale environment rather than a library.
- fredrik-johansson/mpmath — arbitrary-precision *numeric* math only; use it when you need precision, not symbols.
- JuliaSymbolics/Symbolics.jl — Julia CAS built for performance and equation-based modeling; use it when you are in the Julia ecosystem.
- Maxima and Wolfram Mathematica (not primarily on GitHub) — mature standalone CASs; use them when symbolic capability trumps Python integration or open licensing.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2005 | Project started by Ondřej Čertík; core rewritten (~10–100× faster) by Pearu Peterson in 2007[^2]. |
| — | 2007+ | GSoC participation every year since; most development has come through it[^2]. |
| 1.0 | 2016-03 | First stable release; mpmath split out as an external dependency[^5]. |
| 1.5 | 2019-12 | Last release supporting Python 2. |
| 1.6 | 2020-05 | Python 3 only. |
| 1.12 | 2023-05 | Incremental release; deprecation-heavy cleanup cycle. |
| 1.13 | 2024-07 | `Float`/exact-number `==` semantics change; ecosystem breakage[^6]. |
| 1.14 | 2025 | Current stable series. |

## References

[^1]: Meurer et al., "SymPy: symbolic computing in Python", PeerJ Computer Science 3:e103, 2017. https://doi.org/10.7717/peerj-cs.103
[^2]: SymPy README, "Brief History". https://github.com/sympy/sympy#brief-history
[^3]: SymPy documentation, "Gotchas and Pitfalls". https://docs.sympy.org/latest/explanation/gotchas.html
[^4]: SymEngine — fast C++ symbolic manipulation library with SymPy-compatible bindings. https://github.com/symengine/symengine
[^5]: mpmath — Python library for arbitrary-precision floating-point arithmetic. https://mpmath.org/
[^6]: SymPy release notes for 1.13.0. https://github.com/sympy/sympy/wiki/release-notes-for-1.13.0

## Tags

python, computer-algebra, symbolic-math, mathematics, scientific-computing, calculus, equation-solving, code-generation, latex, numfocus, library
