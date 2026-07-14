# python-attrs/attrs

> Python classes without boilerplate — generates `__init__`, `__repr__`, `__eq__`, and friends from declared attributes, and the project that `dataclasses` grew out of.

[GitHub repo](https://github.com/python-attrs/attrs) ·
[Official website](https://www.attrs.org/) ·
[License: MIT](https://github.com/python-attrs/attrs/blob/main/LICENSE)

## Overview

*attrs* is a class-building library: you declare a class's attributes once, and a decorator writes the object protocol methods (initializer, `__repr__`, equality, hashing, optionally ordering) for you. It predates and directly inspired the standard library's `dataclasses` — `dataclasses` (PEP 557, Python 3.7) is described by its own authors as a descendant of *attrs*, and *attrs*'s maintainer was involved in that PEP[^1]. The relationship matters for choosing between them: `dataclasses` is the stdlib subset; *attrs* is the superset that kept iterating.

The library carries two public API surfaces on purpose. The **classic** API (`@attr.s`, `attr.ib()`, imported from the `attr` package) dates to 2015. The **modern** API (`@define`, `@frozen`, `field()`, imported from the `attrs` package) was introduced in 20.1.0 with saner defaults, and the `attrs` import name was added in 21.3.0[^2]. Both are supported indefinitely and are the same machinery underneath — this dual-namespace situation is the single most common source of confusion for newcomers, who find tutorials mixing `attr.ib` and `attrs.field` with no explanation that they are eras of the same tool.

The defining design choice is that *attrs* generates real Python methods and `exec`s them at class-creation time, rather than dispatching through `__getattr__` or metaclass magic at call time. The cost is paid once, when the class is defined; instances then run at hand-written-code speed. This is what lets the project claim no runtime penalty, and it is the same technique `dataclasses` uses. The repository is actively maintained (last push July 2026) and widely depended upon — it sits in the transitive dependency tree of a large fraction of the Python ecosystem.

## Getting Started

```bash
pip install attrs
```

```python
from attrs import define, field, asdict

@define
class Server:
    host: str
    port: int = 8080
    tags: list[str] = field(factory=list)

    @port.validator
    def _check_port(self, attribute, value):
        if not (0 < value < 65536):
            raise ValueError(f"port out of range: {value}")

s = Server("localhost", tags=["web"])
# Server(host='localhost', port=8080, tags=['web'])
assert s == Server("localhost", 8080, ["web"])
asdict(s)   # {'host': 'localhost', 'port': 8080, 'tags': ['web']}
```

Types are optional. Without annotations, assign `field()` directly (`port = field(default=8080)`) and the class still works. `@frozen` produces an immutable, hashable variant; `attrs.evolve(s, port=9090)` returns a modified copy.

## Architecture / How It Works

At class definition, the decorator collects the declared attributes (from annotations under `@define`, or from `field()`/`attr.ib()` assignments), builds an ordered list respecting inheritance (base-class attributes first), and then **generates source code for each requested method as a string, compiles it, and attaches it** to the class. `attrs.fields(Cls)` exposes the collected `Attribute` metadata. Because the methods are ordinary compiled functions, they show up in tracebacks and can be stepped through in a debugger — a deliberate contrast to runtime-dispatch approaches.

Key mechanics:

- **`@define` defaults to slotted classes** (`slots=True`). This is the biggest behavioral difference from the classic `@attr.s` (which defaults to dict-backed classes) and from `dataclasses` (dict-backed). Slots cut per-instance memory and forbid setting undeclared attributes.
- **`field()`** carries `default`, `factory` (for mutable defaults like `list`), `validator`, `converter`, `kw_only`, `init`, `repr`, `eq`, `order`, `hash`, and arbitrary `metadata`. This declarative surface is where *attrs* substantially exceeds `dataclasses`.
- **Converters run before validators** during `__init__`; both are optional. A converter normalizes the incoming value (e.g. `int`), a validator asserts an invariant and raises on failure. *attrs* ships a `validators` module (`instance_of`, `in_`, `optional`, `matches_re`, and composition helpers).
- **Hooks**: `__attrs_post_init__` runs after generated init; `field_transformer` lets you rewrite the whole attribute list at build time (used by typed-settings and cattrs-style tooling); `__attrs_pre_init__` allows calling `super().__init__`.
- **Type resolution is lazy.** Under `from __future__ import annotations` (PEP 563), annotations are strings; `attrs.fields(Cls).<f>.type` stays a string until you call `attrs.resolve_types(Cls)`. Validators and converters are unaffected, but introspection tools must account for this.

**cattrs** ([python-attrs/cattrs](https://github.com/python-attrs/cattrs)) is the sibling project for structuring/unstructuring *attrs* classes to and from JSON-compatible data. *attrs* itself deliberately does **not** do serialization beyond the shallow `asdict`/`astuple` helpers — those two do not round-trip nested unions, and the docs steer non-trivial serialization to cattrs.

## Production Notes

**Slots are the main footgun of the modern API.** Because `@define` sets `slots=True`:
- You cannot monkeypatch instances or add attributes not declared as fields.
- `functools.cached_property` requires a `__dict__`; *attrs* added explicit support so it works on slotted classes, but hand-rolled caching that writes to `self.__dict__` will fail with `AttributeError`.
- Slotted classes are recreated (a new class object) during the decorator run, so anything captured before decoration (e.g. a reference to the pre-slots class) goes stale. This also historically complicated `super()` in slotted classes; *attrs* injects the `__class__` closure cell to keep zero-arg `super()` working.
- Pickling works, but *attrs* generates `__getstate__`/`__setstate__` to make it work — third-party pickling shims that assume `__dict__` may need adjustment.

Switch to `@define(slots=False)` when any of the above bites; you keep every other feature.

**Classic vs modern default drift.** Beyond slots, `@define` changed several defaults relative to `@attr.s`: attributes are collected from bare annotations (`auto_attribs` auto-detected), `eq` is on and `order` is off by default, and keyword-only fields are easier to opt into. Mixing the two decorators in one codebase produces subtly different classes; pick one API per project.

**Versioning is CalVer, not SemVer.** Releases are `YY.minor.micro` (e.g. 23.1.0, 25.3.0). A number jump does not signal a breaking change the way SemVer would — read the changelog rather than inferring risk from the version delta. *attrs* has been unusually conservative about breakage; the classic API has survived a decade.

**`attrs` vs `attr` install.** The distribution is `attrs` on PyPI and provides both import namespaces. Depending on the older `attr` name still resolves, but pin `attrs>=` in requirements; some ecosystem confusion came from the short window where only `attr` existed.

**Performance.** Class-definition time grows with attribute count because code is generated per class; for the rare program that builds thousands of distinct *attrs* classes at import, this is measurable. Instance construction and attribute access are at native speed. Slotted classes reduce memory meaningfully for high-cardinality instance workloads.

## When to Use / When Not

**Use when:**
- You want validators, converters, or per-field customization that `dataclasses` does not offer.
- You want slots by default for memory-sensitive workloads without writing `__slots__` by hand.
- You need to support Python versions or introspection scenarios where `dataclasses` falls short, or you want a debuggable, generated `__init__`.
- You are pairing with cattrs for typed (de)serialization.

**Avoid when:**
- The stdlib `dataclasses` already covers your needs and you want zero third-party dependencies.
- You need parsing/coercion and rich validation as the primary job — pydantic is built for that (at higher runtime cost).
- You want maximum (de)serialization throughput for schemas — msgspec is faster and includes validation.
- You rely heavily on monkeypatching instances and do not want to remember to pass `slots=False`.

## Alternatives

- python/cpython (`dataclasses`) — use instead when the stdlib subset suffices and you want no dependency; you give up validators, converters, and slots-by-default.
- pydantic/pydantic — use instead when runtime data validation, coercion, and schema/JSON serialization are the core requirement, not just class boilerplate.
- jcrist/msgspec — use instead when you need very fast serialization plus validation and can accept a narrower, struct-oriented API.
- python-attrs/cattrs — not a replacement but the companion; add it when you need to structure/unstructure *attrs* classes to and from JSON-like data.

## History

| Version | Date | Notes |
|---------|------|-------|
| 15.0.0 | 2015-04 | First PyPI release under the `attrs` name; classic `@attr.s` / `attr.ib` API. |
| 16.x | 2016 | Validators and converters matured. |
| 17.x | 2017 | Slots support, `attr.evolve`, richer validators. |
| 18.x | 2018 | `dataclasses` (PEP 557) ships in Python 3.7, derived from *attrs*[^1]. |
| 19.x | 2019 | `auto_attribs` type-annotation collection stabilized. |
| 20.1.0 | 2020-08 | Modern next-gen APIs: `@define`, `@frozen`, `field()`[^2]. |
| 21.3.0 | 2021-12 | `attrs` package import name added alongside `attr`[^2]. |
| 22.x | 2022 | Init-time performance work; validator/converter refinements. |
| 23.x | 2023 | `attrs.resolve_types` and alias/typing improvements. |
| 24.x–25.x | 2024–2026 | Ongoing typing, `field_transformer`, and Python-version support updates. |

## References

[^1]: Hynek Schlawack, "Import *attrs*" — on the lineage from *attrs* to `dataclasses`. https://hynek.me/articles/import-attrs/
[^2]: *attrs* README and "On The Core API Names" — modern APIs introduced in 20.1.0, `attrs` import name added in 21.3.0. https://www.attrs.org/en/latest/names.html

## Tags

python, dataclasses, boilerplate, class-generation, validation, oop, slots, serialization, code-generation, library
