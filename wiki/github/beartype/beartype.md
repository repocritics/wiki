# beartype/beartype

> A pure-Python runtime type-checker that enforces PEP type hints in near-constant time by code-generating per-function checkers and sampling containers rather than scanning them.

[GitHub repo](https://github.com/beartype/beartype) ·
[Official website](https://beartype.readthedocs.io) ·
[License: MIT](https://github.com/beartype/beartype/blob/main/LICENSE)

## Overview

beartype is a runtime type-checker for Python: you annotate functions, classes,
and assignments with standard PEP type hints, and beartype enforces them while
the program runs. It occupies the gap between static checkers like mypy (which
never run your code) and hand-written `isinstance` guards (which don't scale to
`list[dict[str, tuple[int, ...]]]`). Decorate a callable with `@beartype` and it
raises a precise, human-readable exception the moment a parameter or return value
violates its annotation.

The project's defining decision is its performance model. Rather than
recursively validating every element of a container — which is O(n) and would
make checking a million-item list unusable — beartype checks a single
pseudo-randomly selected element per nested container per call, in amortized
O(1) time[^1]. This is the central tradeoff: type violations buried deep in a
large collection may not be caught on every call. beartype trades exhaustive
guarantees for a cost low enough to leave enabled in production. Reviewers
should understand this is a statistical enforcement model, not a proof.

The second defining trait is implementation strategy. `@beartype` does not
interpret hints reflectively at call time; it synthesizes specialized Python
source code for each decorated callable's exact signature, `exec`s it once, and
caches the resulting wrapper[^2]. The generated wrapper contains straight-line
checks with no per-call hint introspection, which is why overhead is typically
in the microsecond range. beartype has no runtime dependencies and is pure
Python — no C extension, no build step.

## Getting Started

```bash
pip install beartype
```

Whole-package enforcement via an import hook (the recommended entrypoint):

```python
# your_package/__init__.py
from beartype.claw import beartype_this_package
beartype_this_package()   # every annotated def/class in this package is checked
```

Or decorate a single callable explicitly:

```python
from beartype import beartype

@beartype
def scale(values: list[float], factor: float) -> list[float]:
    return [v * factor for v in values]

scale([1.0, 2.0], 3.0)          # ok
scale(["a", "b"], 3.0)          # raises beartype.roar.BeartypeCallHintParamViolation
```

Ad-hoc checks anywhere, without a decorator:

```python
from beartype.door import is_bearable, die_if_unbearable
is_bearable([1, 2, 3], list[int])          # -> True
die_if_unbearable({"k": 1}, dict[str, str])  # raises on violation
```

## Architecture / How It Works

beartype is organized as a set of public subpackages over a large private core:

- **`@beartype`** — the decorator. For each callable it inspects the signature,
  generates a wrapper function as a string of Python that performs only the
  checks that signature requires, compiles it with `exec`, and memoizes it.
  Decoration cost is paid once; call-time cost is the generated checks only.
- **`beartype.claw`** — import hooks (`beartype_this_package`, `beartype_all`,
  `beartype_package`) that install a meta-path finder and rewrite modules' ASTs
  at import time so annotated objects are decorated automatically. This is how
  beartype covers code you didn't hand-decorate, including whole dependency
  trees, with per-package `violation_type` control (raise in your code, warn in
  others').
- **`beartype.door`** ("Decidable Object-Oriented Runtime-checker") — a
  procedural and object API for checking arbitrary objects against hints outside
  the decorator: `is_bearable`, `die_if_unbearable`, and a `TypeHint` wrapper
  that makes hints introspectable and comparable (subhint relations).
- **`beartype.vale`** — `Is[...]` validators that attach arbitrary boolean
  predicates to a hint via `typing.Annotated`, so constraints like "non-empty"
  or "positive" become part of the type.
- **`beartype.roar`** — the exception and warning hierarchy. Violations carry a
  message pinpointing the offending value's location within a nested structure.

The container-sampling strategy lives beneath all of these: whatever the entry
point, a deeply nested hint compiles into checks that descend a bounded number
of levels and sample one element per level rather than iterating. Hint support
tracks the PEP standards — 484, 544 (protocols), 585 (builtin generics), 586
(`Literal`), 593 (`Annotated`), 604 (`X | Y` unions), 646, and others — with the
private core carrying separate handling paths per PEP. Coupling to CPython
internals is real: beartype does frame and code-object manipulation to produce
useful tracebacks, and each new Python minor version tends to require porting
work.

## Production Notes

- **Sampling is not exhaustive.** `is_bearable(huge_list, list[int])` does not
  guarantee every item is an `int`; it checks a random one. For validating
  untrusted input at a trust boundary (deserialization, request bodies), a
  coercing validator like pydantic or msgspec that inspects every field is the
  correct tool. beartype is for enforcing internal contracts cheaply, not for
  sanitizing external data.
- **Decoration is not free; calls are cheap.** The one-time codegen + `exec` per
  callable adds import-time cost. For packages with very large API surfaces the
  aggregate import-time overhead is measurable; call-time overhead usually is
  not.
- **Pre-1.0 versioning.** beartype remains on the 0.x line and its own tracker
  keeps 1.0.0 open[^3]. The public API has been stable in practice, but minor
  releases have introduced stricter checking that surfaces latent hint bugs in
  downstream code — an upgrade can start raising on annotations that were
  quietly wrong before. Pin the version and read release notes before bumping.
- **Import hooks and startup order.** `beartype_this_package()` must run early
  (top of `__init__`) to catch submodules; AST rewriting only affects modules
  imported after the hook is installed. Mixing `beartype.claw` with other
  import-time machinery (other meta-path finders, lazy importers) can produce
  order-dependent behavior.
- **Third-party hint edge cases.** Exotic hints from libraries (some NumPy,
  Pandas, or SQLAlchemy typing constructs) may be unsupported or checked
  loosely; beartype generally degrades to a permissive check rather than
  erroring, but the behavior is worth confirming for hint-heavy dependencies.

## When to Use / When Not

**Use when:**
- You want your existing PEP type hints enforced at runtime during development
  and testing, catching contract violations that static analysis misses.
- You want cheap, always-on internal invariant checks across a whole package via
  one import hook.
- You need `isinstance`-style checks against complex parametrized hints
  (`is_bearable`) that stdlib `isinstance` cannot express.

**Avoid when:**
- You are validating or coercing untrusted external data — use a boundary
  validator that inspects every field.
- You need a completeness guarantee that every element of every container is
  checked on every call.
- You want zero runtime footprint and are satisfied enforcing types only in CI
  with a static checker.

## Alternatives

- agronholm/typeguard — the closest direct competitor; runtime checker with a
  decorator and import hook. Use it instead when you want more exhaustive
  container checking and can accept higher per-call cost.
- python/mypy — static type checker. Use instead when you want type errors
  caught at CI time with no runtime cost and no runtime enforcement.
- microsoft/pyright — fast static checker with strong editor integration. Use
  for interactive/IDE feedback rather than runtime guarantees.
- pydantic/pydantic — data-model validation and coercion at boundaries. Use when
  you need to parse and normalize untrusted input, not enforce annotations.
- jcrist/msgspec — schema validation tied to fast (de)serialization. Use at IO
  boundaries where you validate every field while decoding.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-04-03 | Public GitHub repo opened; core extracted from the BETSE codebase[^4]. |
| 0.10 | 2022 | DOOR API (`beartype.door`, `is_bearable`, `TypeHint`) introduced. |
| 0.15 | 2023 | `beartype.claw` import hooks for package-wide auto-decoration. |
| 0.x | ongoing | Continued PEP coverage and per-CPython-version porting; still pre-1.0[^3]. |
| — | 2026-07 | Actively maintained; frequent commits on `main`. |

## References

[^1]: beartype documentation, "Nobody Expects the Linearithmic Time" / math and FAQ on constant-time checking. https://beartype.readthedocs.io/en/latest/math/
[^2]: beartype FAQ, on decorator-generated wrapper code. https://beartype.readthedocs.io/en/latest/faq/
[^3]: beartype issue #7, "1.0.0" release tracker. https://github.com/beartype/beartype/issues/7
[^4]: BETSE, the simulator project from which beartype was extracted. https://github.com/betsee/betse

## Tags

python, runtime-type-checking, static-typing, type-hints, pep-484, decorator, validation, import-hooks, pure-python, developer-tools
