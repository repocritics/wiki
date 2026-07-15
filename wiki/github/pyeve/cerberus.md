# pyeve/cerberus

> Schema-as-data validation for Python: you validate documents against dicts, not against typed classes.

[GitHub repo](https://github.com/pyeve/cerberus) ·
[Official website](http://python-cerberus.org) ·
[License: ISC](https://github.com/pyeve/cerberus/blob/1.3.x/LICENSE)

## Overview

Cerberus is a dependency-free data validation and normalization library for
Python, written by Nicola Iarocci[^1]. It began life as the validation layer of
Eve, Iarocci's REST API framework, and was extracted into a standalone package
that Eve still consumes[^2]. The repository dates to 2012; it reached a stable
1.x line and has since been in low-frequency maintenance rather than active
feature development.

Its defining characteristic is that a schema is ordinary data — a Python `dict`
(readily loaded from YAML or JSON) that maps field names to rule sets. This is
the axis on which it should be judged. When schemas are dynamic — supplied by
users, stored in a database, assembled at runtime, or shared across services in
a language-neutral file — Cerberus fits naturally, because the schema is a value
you can build, merge, and serialize. The cost is everything a class-based
validator gives you for free: no static types, no editor autocomplete on
validated data, no `mypy` inference, and pure-Python speed rather than a
compiled core.

That tradeoff frames the ecosystem decision. Since pydantic v2 shipped a Rust
core and became the default for typed request/response models, most new projects
that want typed models reach for it. Cerberus remains the straightforward answer
when the schema itself is the runtime input, not source code, and when you want
validation and normalization in one pass without adopting a type-annotation
model layer.

## Getting Started

```console
$ pip install cerberus
```

Cerberus has no runtime dependencies[^3].

```python
from cerberus import Validator

schema = {
    "name": {"type": "string", "required": True, "maxlength": 50},
    "age":  {"type": "integer", "min": 0, "coerce": int},
    "role": {"type": "string", "allowed": ["admin", "user"], "default": "user"},
}

v = Validator(schema)
v.validate({"name": "John Doe", "age": "30"})   # True
v.document        # {'name': 'John Doe', 'age': 30, 'role': 'user'}
v.errors          # {}

v.validate({"age": -1})
v.errors          # {'name': ['required field'], 'age': ['min value is 0']}
```

`validate()` returns a bool and never raises on invalid data — errors accumulate
in `.errors` as a nested tree. `validated(doc)` returns the normalized document
or `None`. Note the two effects above: `age` was coerced from string to int, and
`role` was filled from its default — normalization happens alongside validation.

## Architecture / How It Works

A `Validator` is constructed with a schema and, on first use, checks that schema
against an internal meta-schema — so a malformed rule set is itself reported as
an error rather than failing cryptically later. Each rule name (`type`,
`required`, `min`, `regex`, `allowed`, `schema`, …) maps by naming convention to
a `_validate_<rule>` method; each type (`_validate_type_<name>`) likewise. This
introspection is what makes the library "easily extensible" — you subclass
`Validator`, add methods, and the new rules are discovered automatically. A rule
method's own constraints live in its docstring as an embedded schema fragment,
an unusual pattern that keeps rule definition and rule metadata together.

Validation and **normalization** are distinct phases. Normalization applies
`coerce` (transform functions), `default` / `default_setter`, `rename` /
`rename_handler`, and `purge_unknown` before or during rule checks. Ordering
matters: a `coerce` that runs before type-checking can convert a value into a
valid one — or mask what would otherwise be a type error — so coercers should be
written defensively.

Reusable definitions go into `schema_registry` and `rules_set_registry`, letting
named sub-schemas be referenced by string. Nested structures use `schema` (for
mappings and, with `type: list`, for each item), while `keysrules` /
`valuesrules` constrain arbitrary-keyed dicts. Logical combinators
(`anyof`, `allof`, `oneof`, `noneof`) and cross-field rules (`dependencies`,
`excludes`, `requires`) compose more complex constraints. Errors are produced by
a pluggable error handler — `BasicErrorHandler` yields the human-readable dict
seen above; swap it to emit structured output for an API.

## Production Notes

- **A `Validator` instance is not thread-safe.** It carries per-validation state
  (`document`, `errors`, current path). Do not share one instance across
  threads or concurrent requests; construct per call, or one per thread. The
  schema itself is cheap to reuse — pass it into a fresh `Validator`, or use the
  registries to avoid re-parsing.
- **It is pure Python and comparatively slow.** For high-throughput hot paths,
  a compiled-core validator (pydantic v2) will be materially faster. Cerberus is
  well-suited to config files, moderate-rate request payloads, and batch jobs;
  it is a poor fit for validating millions of records in a tight loop.
- **Schema meta-validation has a first-use cost.** Reuse validators / registries
  rather than rebuilding schemas on every request so the check is not repaid.
- **`keyschema` / `valueschema` were renamed** to `keysrules` / `valuesrules` in
  the 1.3 line; the old names were deprecated and then removed. Schemas copied
  from older tutorials silently fail meta-validation on current versions[^4].
- **Unknown fields are rejected by default.** `allow_unknown` permits them (and
  can itself carry a rule set); `purge_unknown` drops them during normalization.
  Getting these backwards is a common source of "my valid document keeps
  failing" or "unexpected keys leaked through."
- **No typing payoff.** The validated document is a plain `dict` with no static
  type information — none of the IDE/`mypy` benefits a class-based model gives.
  If your schema is effectively static, prefer a typed model layer instead.

## When to Use / When Not

**Use when:**
- Schemas are dynamic — user-supplied, DB-stored, config-driven, or shared as
  language-neutral YAML/JSON.
- You want validation and normalization (coercion, defaults, renaming, purging)
  in a single pass.
- You need custom rules or types and prefer a subclass-and-register model.
- You want zero dependencies and a small, stable surface.

**Avoid when:**
- Your models are static and known in code — a typed model layer gives you
  autocomplete, `mypy`, and speed.
- You are on a throughput-critical hot path where a Rust-core validator wins.
- You want serialization/deserialization and rich type ecosystems bundled in;
  Cerberus validates and normalizes, it is not an ORM/serializer.

## Alternatives

- pydantic/pydantic — class-based, type-annotated models with a Rust core; the
  default when schemas are static Python and speed matters.
- python-jsonschema/jsonschema — use when you must conform to the JSON Schema
  standard for cross-language interop rather than Cerberus's own dialect.
- marshmallow-code/marshmallow — use when you need serialization/deserialization
  and object marshalling, not just validation.
- alecthomas/voluptuous — use for a similarly dependency-light, data-driven
  validator with a callable/predicate-composition style.
- keleshev/schema — use for very small, expression-style schema checks where
  Cerberus's rule vocabulary is more than you need.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2012–2016 | Extracted from Eve; long pre-1.0 series[^2]. |
| 1.0 | 2016 | First stable release. |
| 1.2 | 2018 | Semantic versioning adopted from this release on[^5]. |
| 1.3 | 2019 | `keyschema`/`valueschema` renamed to `keysrules`/`valuesrules`[^4]. |
| 1.3.x | 2019–2026 | Current maintenance branch; low-frequency fixes and interpreter support updates. |

## References

[^1]: "Cerberus is an open source project by Nicola Iarocci." — project README and https://nicolaiarocci.com/
[^2]: Eve (pyeve/eve), the REST framework Cerberus was extracted from and which still uses it. https://github.com/pyeve/eve
[^3]: "It has no dependencies" — Cerberus README, Features section. https://github.com/pyeve/cerberus
[^4]: Cerberus documentation, validation rules (`keysrules` / `valuesrules`). http://docs.python-cerberus.org
[^5]: "Starting with Cerberus 1.2, it is maintained according to semantic versioning." — Cerberus README, Versioning section.

## Tags

python, data-validation, schema-validation, normalization, validation-library, dependency-free, runtime-schema, config-validation
