# python-attrs/cattrs

> Composable converters that (un)structure and validate Python data — without putting serialization logic inside your models.

[GitHub repo](https://github.com/python-attrs/cattrs) ·
[Official website](https://catt.rs) ·
[License: MIT](https://github.com/python-attrs/cattrs/blob/main/LICENSE)

## Overview

cattrs converts between "unstructured" data (dicts, lists, primitives — the shape you get from `json.loads`) and "structured" data (typed `attrs` classes, dataclasses, `TypedDict`, `NamedTuple`, enums, and standard-library generics). Its two verbs are `structure(data, Type)` and `unstructure(obj)`. It was created by Tin Tvrtković (@Tinche) and lives under the `python-attrs` organization alongside `attrs` itself[^1].

The defining design decision is that **un/structuring rules are separate from the models**[^2]. Unlike Pydantic — where validation and serialization live *on* a `BaseModel` you inherit from — cattrs keeps your data classes plain and describes conversion externally, in a `Converter`. This gives a one-to-many relationship: the same class can be structured differently for JSON, msgpack, or an internal API, and you can register rules for types you do not own and cannot edit. The tradeoff is a second concept to hold in your head (the converter and its hook registry) rather than one annotated class.

The second decision is "resist the temptation to guess." Where a mapping is ambiguous (which member of a union? how to disambiguate?), cattrs refuses a default and requires explicit configuration rather than picking silently. This makes it verbose for irregular schemas but predictable for large ones.

## Getting Started

```bash
pip install cattrs
```

```python
from attrs import define
from cattrs import structure, unstructure

@define
class Point:
    x: int
    y: int

@define
class Line:
    start: Point
    end: Point

data = {"start": {"x": 0, "y": 0}, "end": {"x": 3, "y": 4}}
line = structure(data, Line)          # Line(start=Point(0, 0), end=Point(3, 4))
back = unstructure(line)              # {"start": {"x": 0, "y": 0}, ...}
```

Nested `attrs`/dataclass structures work out of the box because the field types carry the recursion. For a JSON pipeline that also handles `datetime`, `set`, and `bytes`, use a preconfigured converter:

```python
from cattrs.preconf.orjson import make_converter
conv = make_converter()
conv.dumps(line)                      # bytes, via orjson
conv.loads(raw, Line)
```

## Architecture / How It Works

The unit of behavior is a `Converter`, which holds two registries of **hooks** — one for structuring, one for unstructuring — keyed by type. `structure`/`unstructure` at module scope are just methods on a global converter instance.

For an `attrs` class or dataclass, cattrs does not reflect over the fields on every call. On first encounter it **generates a specialized Python function** for that exact class (via `make_dict_structure_fn` / `make_dict_unstructure_fn`), compiles it, caches it on the converter, and reuses it thereafter[^3]. This code-generation approach is the source of cattrs' speed: the per-instance path is straight-line attribute access with no runtime introspection. The generating converter was originally a separate `GenConverter`; in the 22.1 release it became the default `Converter` and CalVer versioning was adopted[^4].

Hooks are resolved most-specific-first. Beyond exact-type registration (`register_structure_hook`), predicate-based **hook factories** (`register_structure_hook_factory`, `register_unstructure_hook_func`) let you match by an arbitrary function over the type, which is how generic containers, unions, and `Annotated` metadata are handled.

**Unions** are the hard case. Structuring `A | B` of attrs classes uses their unique required fields to disambiguate by default; when that is ambiguous cattrs raises rather than guesses. For reliability the recommended pattern is the tagged-union strategy (`configure_tagged_union`) or subclass handling (`include_subclasses`), which write an explicit discriminator field[^5].

**Detailed validation** (default since 22.1) runs every field and aggregates failures into an `ExceptionGroup` — `ClassValidationError` / `IterableValidationError` nesting the individual errors, with `transform_error` to flatten them into a list of human-readable strings. cattrs deliberately reused the standard library's exception-group mechanism instead of inventing its own error type[^2].

**preconf** converters (`cattrs.preconf.{json,orjson,ujson,msgpack,cbor2,bson,pyyaml,tomlkit,msgspec}`) preload hooks for the types each format cannot represent natively, so `datetime`, `set`, `bytes`, and `Enum` round-trip correctly per backend.

## Production Notes

- **Create your own converter for custom hooks.** Registering hooks on the module-global converter mutates process-wide behavior and is a common source of action-at-a-distance bugs. Instantiate a `Converter()` (or a preconf one) per serialization context.
- **Hook generation is cached; converter mutation is not free.** The first structure of a class pays a one-time codegen cost. If you register a hook *after* a class's hook has already been generated and cached, the cached function may not pick it up as expected — configure the converter before first use.
- **Detailed validation costs speed.** The aggregating-exception-group path is slower than fail-fast. For hot paths where you trust the input, construct the converter with `detailed_validation=False` to get a plain first-error exception and a faster path.
- **String annotations can fail to resolve.** With `from __future__ import annotations` (PEP 563) or forward references, cattrs resolves types via `get_type_hints`; classes defined in local scopes or with unresolvable names raise at structure time, not definition time.
- **It is fast, but not msgspec-fast.** cattrs' codegen closes most of the gap with hand-written converters, but msgspec (C extension) is measurably faster for raw decode throughput. cattrs' advantage is composability and model-agnosticism, not being the fastest decoder.
- **Best with attrs; workable with dataclasses; manual for plain classes.** `attrs` classes get the richest support (field metadata, `Annotated` handling). Dataclasses, `TypedDict`, and `NamedTuple` are supported; arbitrary hand-written classes need explicit hooks.
- **Python version support moves forward.** cattrs drops end-of-life Python versions on a rolling basis; pin a compatible range if you are on an older interpreter.

## When to Use / When Not

**Use when:**
- You already use `attrs` or dataclasses and want to keep serialization out of your models.
- You need the *same* type serialized multiple ways, or must convert types you don't own.
- You want validation errors aggregated (all failures at once) via exception groups.
- You want format-specific correctness (datetime/bytes/set) across JSON, msgpack, cbor, YAML from one library.

**Avoid when:**
- You want validation and the model definition coupled in one class with a large ecosystem — Pydantic fits that better.
- Raw decode throughput is the bottleneck and you can adopt a dedicated struct type — msgspec is faster.
- Your data is highly irregular/ambiguous — cattrs' no-guessing stance means a lot of manual disambiguation.
- You want zero extra concepts beyond your class — the converter/hook model is real surface area.

## Alternatives

- pydantic/pydantic — validation and serialization *on* the model via `BaseModel`; use it when you want one coupled class and the FastAPI ecosystem, and don't mind inheriting a base.
- jcrist/msgspec — C-based `Struct` types with very fast JSON/msgpack; use it when decode speed dominates and its model type is acceptable.
- marshmallow-code/marshmallow — explicit `Schema` classes separate from models; use it when you want schema objects and pre-3.7-style workflows.
- lidatong/dataclasses-json — dataclass-scoped JSON via decorator/mixin; use it for a small dataclass-only project without cattrs' converter model.
- python-attrs/attrs — the class library cattrs complements (defines the models, not the conversion); use together, not instead.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2016 | Initial PyPI release; structure/unstructure for attrs classes[^1]. |
| 1.0.0 | 2019 | First stable release. |
| 22.1.0 | 2022-01 | CalVer adopted; generating converter becomes default `Converter`; detailed validation via exception groups[^4]. |
| 22.2.0 | 2022-10 | `TypedDict` support, tagged-union strategy improvements. |
| 23.1.0 | 2023-05 | `typing.NewType`, PEP 695 groundwork, disambiguation improvements. |
| 23.2.0 | 2023-11 | Broader typing/`Annotated` support, preconf refinements. |
| 24.1.0 | 2024 | Continued typing coverage (PEP 695 type aliases on 3.12+). |

## References

[^1]: cattrs project, `python-attrs` organization. https://github.com/python-attrs/cattrs
[^2]: cattrs documentation, "Design Decisions" / README. https://catt.rs/en/stable/
[^3]: cattrs documentation, "Customizing (un-)structuring" — generated dict hooks. https://catt.rs/en/stable/customizing.html
[^4]: cattrs History / changelog — 22.1.0 (CalVer, default generating converter, detailed validation). https://catt.rs/en/stable/history.html
[^5]: cattrs documentation, "Strategies" — tagged unions and subclasses. https://catt.rs/en/stable/strategies.html

## Tags

python, serialization, deserialization, attrs, dataclasses, validation, type-hints, json, converters, data-modeling
