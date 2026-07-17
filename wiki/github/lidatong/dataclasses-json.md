# lidatong/dataclasses-json

> A decorator/mixin that bolts JSON serialization onto stdlib dataclasses, using marshmallow underneath for the schema path.

[GitHub repo](https://github.com/lidatong/dataclasses-json) ·
[Documentation](https://lidatong.github.io/dataclasses-json) ·
[License: MIT](https://github.com/lidatong/dataclasses-json/blob/master/LICENSE)

## Overview

`dataclasses-json` adds `to_json`/`from_json`/`to_dict`/`from_dict` to plain
[PEP 557](https://peps.python.org/pep-0557/) dataclasses via either a
`@dataclass_json` class decorator or a `DataClassJsonMixin` base[^1]. It was
created in 2018[^2], shortly after dataclasses landed in Python 3.7, to fill the
gap the standard library left open: `dataclasses.asdict` exists, but there is no
built-in inverse (`dict` → typed instance), no JSON string handling, and no name
remapping. This library supplies all three with almost no boilerplate.

The defining tension is that it exposes two serialization paths with different
guarantees. The fast path — `from_dict`/`from_json` — does **no type
validation**: `Person.from_json('{"name": 42}')` happily builds an instance with
an `int` where a `str` was declared, because dataclass construction itself does
not check types[^1]. The strict path — `.schema().loads(...)` — generates a
[marshmallow](https://marshmallow.readthedocs.io) schema that *does* validate and
raises `ValidationError`, but is slower and pulls in a heavier dependency. Users
routinely reach for the ergonomic decorator, assume they are getting validation,
and discover otherwise in production.

It has never reached a 1.0 release; the project has lived in the `0.x` range its
entire history and is best characterized as feature-stable and lightly maintained
rather than actively developed. For greenfield code needing validation most of
the ecosystem has moved to pydantic — but dataclasses-json remains attractive
precisely because it keeps your models as *real* stdlib dataclasses rather than a
framework's base class.

## Getting Started

```bash
pip install dataclasses-json
```

```python
from dataclasses import dataclass
from dataclasses_json import dataclass_json

@dataclass_json      # must sit ABOVE @dataclass — order matters
@dataclass
class Person:
    name: str

Person(name="lidatong").to_json()          # '{"name": "lidatong"}'
Person.from_json('{"name": "lidatong"}')   # Person(name='lidatong')

# validation only on the schema path:
Person.from_json('{"name": 42}')           # OK — no type check
Person.schema().loads('{"name": 42}')      # raises marshmallow ValidationError
```

## Architecture / How It Works

The library has two layers that are easy to conflate:

1. **The decorator/mixin layer** does structural (de)serialization by walking a
   dataclass's `__dataclass_fields__` and type annotations recursively. It maps
   JSON scalars and collections onto typed fields, recurses into nested
   dataclasses, and applies per-field `config(...)` metadata (encoders, decoders,
   `field_name` overrides, `LetterCase`). This path never touches marshmallow and
   never validates types — it is essentially a typed `asdict`/`fromdict` with
   hooks.
2. **The `.schema()` layer** lazily generates a marshmallow `Schema` equivalent to
   one you would hand-write, plus an object hook so `load` returns your dataclass
   instead of a `dict`[^3]. This is where validation, `many=True`, and
   marshmallow field customization (`mm_field`) live.

Type resolution reads `__annotations__` directly. A consequence documented in the
README is that `from __future__ import annotations` (PEP 563 string-ized
annotations) **breaks** the library, because it then sees strings instead of
resolved type objects[^1]. Forward references for recursive models must instead
use the quoted-string form (`Optional['Tree']`).

Field-level behavior rides in `dataclasses.field(metadata=config(...))`, nested
under a namespaced key so it coexists with other metadata users. Global codec
extension mutates `dataclasses_json.cfg.global_config.encoders`/`decoders` — a
convenient but process-wide, shared-mutable registry.

Default codecs have surprising semantics: `datetime` encodes to a **float POSIX
timestamp**, not ISO 8601, and a naive datetime is assumed system-local on the
way out and decoded back to a *tz-aware* object — so encode/decode is
deliberately **not** a strict round-trip unless you override the codec[^1].
`UUID` and `Decimal` both encode to `str`.

## Production Notes

- **The marshmallow coupling is the main upgrade hazard.** The strict path and
  `.schema()` depend on marshmallow 3.x; marshmallow 4 shipped breaking changes,
  and dataclasses-json constrains its dependency accordingly[^4]. If another
  package in your environment wants marshmallow 4, you can hit an unsatisfiable
  resolver conflict. Pin deliberately.
- **`.schema()` is not cached.** Every call regenerates the schema. For nested or
  hot-path models, hoist it into a module-level variable (`SCHEMA =
  Person.schema()`) and reuse it, or the schema-build cost dominates[^1].
- **Silent non-validation.** Malformed upstream data produces
  structurally-valid-but-wrong objects that fail much later. Go through
  `.schema()` for input validation — note `Undefined.RAISE` guards only *unknown*
  fields, not field *types*.
- **Unknown-field handling is a three-way choice** (`Undefined.RAISE` /
  `EXCLUDE` / `INCLUDE` with a `CatchAll` field) whose semantics differ subtly
  across `from_dict`, `schema().load`, and `__init__`[^1]. Standardize on one.
- **`LetterCase` assumes snake_case source fields.** Applying `LetterCase.CAMEL`
  to non-snake_case fields yields undefined behavior and quiet mismatches[^1].
  numpy/pandas types are likewise unsupported without explicit codec hooks.
- **Maintenance cadence is slow.** A long tail of open issues and infrequent
  releases; treat it as stable-and-quiet, not a source of rapid edge-case fixes.

## When to Use / When Not

**Use when:**
- You want your models to stay ordinary stdlib dataclasses (for interop, typing,
  or to avoid a framework base class) and only need light JSON glue.
- Serialization ergonomics matter more than runtime validation.
- You are already invested in marshmallow and want schemas auto-generated from
  dataclasses.

**Avoid when:**
- You need real runtime type validation or coercion as a first-class guarantee —
  reach for pydantic or msgspec instead.
- Serialization is performance-critical (both paths are pure-Python; the schema
  path is heavier still).
- You want an actively evolving library with a 1.0 stability contract and fast
  issue turnaround.

## Alternatives

- pydantic/pydantic — validation-first models with a Rust core in v2; use when
  runtime validation, coercion, and speed are the point, and a base class is fine.
- jcrist/msgspec — very fast serialization plus validation for msgpack/JSON; use
  when throughput dominates and you can accept its type model.
- python-attrs/cattrs — composable structure/unstructure converters for attrs and
  dataclasses; use when you want explicit, extensible conversion without marshmallow.
- konradhalas/dacite — minimal dict→dataclass builder, no serialization or extra
  deps; use when you only need decoding and want the smallest surface.
- marshmallow-code/marshmallow — the underlying schema engine; use directly when
  you want full manual control over schemas and validation.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2018-04-21 | Repo created shortly after dataclasses shipped in 3.7[^2]. |
| 0.5.x | 2020–2022 | Long-lived line; `Undefined`/`CatchAll`, `LetterCase`, field overrides matured. |
| 0.6.x | 2023– | Dropped older Pythons; current line. Still pre-1.0. |

*(Point-release dates approximate — see GitHub Releases for exact tags; no 1.0
has ever shipped.)*

## References

[^1]: Project README and documentation. https://github.com/lidatong/dataclasses-json/blob/master/README.md
[^2]: GitHub repository metadata, repo created 2018-04-21. https://github.com/lidatong/dataclasses-json
[^3]: marshmallow schema API. https://marshmallow.readthedocs.io/en/stable/api_reference.html
[^4]: PyPI project dependencies (marshmallow 3.x constraint). https://pypi.org/project/dataclasses-json/

## Tags

python, json, serialization, dataclasses, deserialization, marshmallow, schema, data-modeling, stdlib
