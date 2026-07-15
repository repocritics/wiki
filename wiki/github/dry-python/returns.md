# dry-python/returns

> Typed functional-programming containers for Python — Maybe, Result, IO, and Future — enforced by a mypy plugin.

[GitHub repo](https://github.com/dry-python/returns) ·
[Official website](https://returns.rtfd.io) ·
[License: BSD-2-Clause](https://github.com/dry-python/returns/blob/master/LICENSE)

## Overview

`returns` brings a small, opinionated set of functional-programming primitives to Python: `Maybe` (no more `None`), `Result` (no more raised exceptions in your control flow), `IO`/`IOResult` (impurity marked in the type), `Future`/`FutureResult` (the same for `async` code), and `RequiresContext` (typed dependency injection).[^1] It is authored by Nikita Sobolev (`sobolevn`) under the `dry-python` organization, and is part of the same ecosystem as `wemake-python-styleguide`.[^2] The repository dates to 2019 and remains actively maintained as of 2026.

The defining bet of the library is not the containers themselves — those are a few hundred lines each — but the **mypy plugin**. Python has no native higher-kinded types (HKT), no `do`-notation, and no way to express "a function generic over any container." `returns` emulates all of these, and the emulation only fully type-checks when its bundled mypy plugin is enabled.[^3] This is the central tension: the value proposition is *type safety*, but that safety is delivered by a specific type checker running a specific plugin, not by the language. Turn the plugin off, or use a checker that cannot load it, and much of the guarantee degrades to runtime behavior with weaker static coverage.

The audience is teams that already want Railway Oriented Programming[^4] and explicit effects in Python and are willing to accept verbosity and mypy coupling to get them. It is not a general-purpose utility belt; it is a discipline you opt into across a codebase.

## Getting Started

```bash
pip install returns
# pin a known-good mypy together with the library:
pip install "returns[compatible-mypy]"
```

Enable the plugin (required for the type guarantees to hold):

```toml
# pyproject.toml
[tool.mypy]
plugins = ["returns.contrib.mypy.returns_plugin"]
```

```python
from returns.result import Result, safe

@safe  # converts a raising function into one returning Result[T, Exception]
def divide(a: int, b: int) -> float:
    return a / b

result: Result[float, Exception] = divide(1, 0)
# => Failure(ZeroDivisionError('division by zero'))

print(result.value_or(0.0))   # 0.0 — no exception ever escapes
```

Composition is done with `flow`/`pipe` and point-free helpers rather than method chains:

```python
from returns.pipeline import flow
from returns.pointfree import bind
from returns.result import safe

@safe
def parse(x: str) -> int:
    return int(x)

flow('10', parse, bind(safe(lambda n: n * 2)))  # Success(20)
```

## Architecture / How It Works

The library is layered:

1. **Containers** — `Maybe`, `Result`, `IO`, `IOResult`, `Future`, `FutureResult`, and the `RequiresContext*` family. Each is an immutable wrapper exposing `.map` (transform the inner value), `.bind` (chain a function that itself returns the container), and container-specific helpers like `.value_or` / `.alt` / `.lash`.
2. **Interfaces / laws** — containers are built from composable typeclass-style interfaces (`Mappable`, `Bindable`, `Applicative`, `Container`, etc.) with associated laws, so you can define your own lawful container and reuse the ecosystem.[^5]
3. **HKT emulation** — a `Kind1`/`Kind2`/`Kind3` encoding lets functions be generic over "any container of shape F," approximating higher-kinded polymorphism that Python's type system does not natively support.[^3]
4. **`do`-notation** — a generator-based syntax (`Result.do(...)`) that flattens nested `.bind` chains into readable sequential code, standing in for the `do`/`for`-comprehension sugar other languages provide.
5. **The mypy plugin** — the load-bearing piece. It teaches mypy about the HKT encoding, refines the return types of decorators like `@safe`, `@maybe`, `@impure`, and `@impure_safe`, and makes point-free `bind`/`map` type-check. Without it, mypy sees the raw generics and reports spurious errors or loses precision.

Decorators bridge ordinary Python and the container world: `@safe` and `@impure_safe` catch exceptions into `Failure`/`IOFailure`; `@maybe` lifts `Optional` returns into `Maybe`; `@impure` marks side-effecting functions as `IO`. `unsafe_perform_io` is the deliberate escape hatch back to raw values, meant only for the top edge of a program.

## Production Notes

**The mypy-plugin coupling is the biggest operational constraint.** The static guarantees are real only under mypy with the plugin loaded. `pyright`/Pylance — the default in VS Code — does not run mypy plugins, so editors backed by it will not understand the HKT machinery and will surface false positives or weaker inference.[^3] Teams standardize on mypy-in-CI to get the intended experience.

**mypy version pinning is a recurring upgrade tax.** Because the plugin hooks into mypy internals, a given `returns` release supports a bounded range of mypy versions; the `returns[compatible-mypy]` extra exists precisely because upgrading mypy independently can break type checking until `returns` catches up. Plan mypy and `returns` bumps together, not separately.

**Verbosity and readability cost.** Python lacks syntax for monadic composition, so real code leans on `flow`, `bind`, `pointfree` wrappers, and `do`-notation generators. Reviewers unfamiliar with FP pay a comprehension cost, and stack traces through combinator layers are harder to read than plain `try/except`.

**Runtime overhead.** Every step allocates a wrapper object and a closure. For hot loops this is measurable; the library's benefit is architectural clarity and type safety, not speed. Keep containers at boundaries, not inside tight numeric kernels.

**It is still pre-1.0.** The project has lived in the `0.x` range for its entire history, and while the core containers are stable in practice, the version number signals that occasional API refinement across minor releases is possible; pin the version and read release notes before upgrading.

**All-or-nothing adoption.** The value comes from a codebase-wide convention. Sprinkling `Result` into an otherwise exception-driven service tends to produce the worst of both — extra ceremony without the guarantee that errors are handled at the type level.

## When to Use / When Not

**Use when:**
- Your team already commits to mypy-in-CI and wants explicit, typed error and effect handling.
- You want Railway Oriented Programming and `None`-free code enforced statically, not by convention.
- You are building libraries or domain logic where "which functions can fail / are impure" should be visible in signatures.
- You want typed functional dependency injection (`RequiresContext`) instead of a DI framework.

**Avoid when:**
- Your editor/checker is `pyright`-based and you cannot move type checking to mypy.
- The team is not invested in FP; idiomatic Python with exceptions will be more readable to them.
- You only need a single `Result` type — the full HKT/plugin apparatus is overkill.
- Performance-critical inner loops dominate and wrapper allocation matters.

## Alternatives

- rustedpy/result — use when you want only an `Ok`/`Err` `Result` type with no mypy plugin, HKT, or effect containers.
- dbrattli/Expression — use when you want a broader F#-inspired toolkit (Option, Result, pipelines, immutable collections) with a more pyright-friendly design.
- dbrattli/OSlash — use for learning monads/functors in Python; more educational than production-oriented.
- pytoolz/toolz — use when you need functional composition over iterables and dicts, not typed error/effect containers.
- Standard library `Optional` + exceptions — use when the team prefers idiomatic Python and does not want a cross-cutting FP dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-01 | Repository created under `dry-python`; early `Result`/`Maybe` containers.[^6] |
| 0.x | 2019–2026 | Long-lived pre-1.0 series: `IO`/`IOResult`, `Future`/`FutureResult`, `RequiresContext` family, HKT emulation, mypy plugin, and `do`-notation added incrementally. |
| (current) | 2026-07 | Still actively maintained on the `master` branch; remains pre-1.0.[^1] |

## References

[^1]: dry-python/returns repository and README. https://github.com/dry-python/returns
[^2]: dry-python organization; sibling project wemake-python-styleguide. https://github.com/wemake-services/wemake-python-styleguide
[^3]: returns docs — HKT and the mypy plugin. https://returns.readthedocs.io/en/latest/pages/hkt.html
[^4]: Scott Wlaschin, "Railway Oriented Programming." https://fsharpforfunandprofit.com/rop/
[^5]: returns docs — create your own container / interfaces. https://returns.readthedocs.io/en/latest/pages/create-your-own-container.html
[^6]: GitHub API metadata, `created_at` 2019-01-26. https://github.com/dry-python/returns

## Tags

python, functional-programming, monads, result-type, railway-oriented-programming, mypy, type-safety, error-handling, dependency-injection, dry-python
