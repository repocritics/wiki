# marshmallow-code/marshmallow

> An ORM/ODM/framework-agnostic Python library for converting complex objects to and from simple native datatypes, with validation.

[GitHub repo](https://github.com/marshmallow-code/marshmallow) ·
[Official website](https://marshmallow.readthedocs.io/) ·
[License: MIT](https://github.com/marshmallow-code/marshmallow/blob/dev/LICENSE)

## Overview

marshmallow is a schema library for serialization (Python objects → primitive types like `dict`/`str`/`int`, ready for JSON), deserialization (incoming primitives → validated app-level data), and validation. You declare a `Schema` with typed fields; `dump()` goes object-to-primitive, `load()` goes primitive-to-object and raises on invalid input. Created by Steven Loria in 2013[^1], it became the default validation layer for the Flask/REST ecosystem and underpins webargs, apispec, and flask-smorest.

Its defining design choice is being **framework- and ORM-agnostic**: schemas are separate classes, decoupled from your domain models. You do not make your SQLAlchemy/Django/attrs objects inherit from anything, and one object can have several schemas (public API view, admin view, ingest). This separation is marshmallow's main advantage over model-first libraries — and the source of its main cost: it is pure Python and comparatively slow.

That cost is the central tension in 2026. pydantic v2 moved its validation core to Rust and, coupled with FastAPI, absorbed most of the "parse and validate request bodies" use case[^2]. marshmallow remains widely deployed where serialization must stay decoupled from the model layer, where the Flask/webargs/apispec stack is already in place, or where multiple divergent representations of one object are needed. It competes on flexibility and ecosystem, not on throughput.

## Getting Started

```bash
pip install -U marshmallow
```

```python
from marshmallow import Schema, fields, ValidationError, post_load

class UserSchema(Schema):
    name = fields.Str(required=True)
    email = fields.Email(required=True)
    age = fields.Int(load_default=None)

# Serialize: object -> primitives
schema = UserSchema()
print(schema.dump({"name": "Tom", "email": "tom@example.com", "age": 40}))
# {'name': 'Tom', 'email': 'tom@example.com', 'age': 40}

# Deserialize + validate: primitives -> data (raises on bad input)
try:
    schema.load({"name": "Tom", "email": "not-an-email"})
except ValidationError as err:
    print(err.messages)  # {'email': ['Not a valid email address.']}
```

By default `load()` returns a `dict`. To get a real object, attach a `@post_load` hook that constructs one from the validated data.

## Architecture / How It Works

A `Schema` subclass is processed by a metaclass that collects every class-level `Field` into a `_declared_fields` mapping; instances resolve that into a bound `fields` dict, honoring `Meta.fields`/`exclude`/`only` and per-instance `only`/`exclude`. Each `Field` implements `_serialize(value, attr, obj)` and `_deserialize(value, attr, data)`; `dump`/`load` iterate fields and delegate. Field construction options — `required`, `allow_none`, `load_default`, `dump_default`, `data_key`, `validate` — are stored on the field, not evaluated until dump/load time.

Validation is layered: individual field validators (`marshmallow.validate.Length`, `Range`, `OneOf`, `Regexp`, or any callable), field-level `@validates("name")` methods, and schema-level `@validates_schema` for cross-field rules. Failures accumulate into a single `ValidationError` whose `.messages` is a nested dict keyed by field — marshmallow collects all errors in a pass rather than failing on the first.

Lifecycle hooks (`@pre_load`, `@post_load`, `@pre_dump`, `@post_dump`, each optionally `pass_many=True`) let you rename keys, coerce shapes, or build objects. `fields.Nested(OtherSchema)` composes schemas for nested structures; `fields.List`/`fields.Dict` handle collections; `many=True` maps a schema over a sequence. Self-referential nesting is expressed by passing the schema by name string or `lambda: SchemaName()`.

Everything is runtime and declarative. Unlike pydantic or dataclasses, a schema carries no static type information about what `load()` returns — to a type checker the result is `Any`/`dict`. This is the recurring DX complaint: the shape enforced at runtime is invisible to editors and mypy.

## Production Notes

**Throughput.** marshmallow is pure Python; serializing large collections or hot API paths can make it the profiler's top line. Teams that hit this either cache dumped output, narrow schemas with `only=`/`exclude=`, or migrate the hot endpoints to pydantic v2. Do not assume `dump()` is free at scale.

**The v2 → v3 break was large.** In v2, `load()`/`dump()` returned a `(data, errors)` tuple (`MarshalResult`); v3 (2019) made them return data directly and *raise* `ValidationError` instead. Every call site that unpacked the tuple broke, and `strict=True` was removed because raising became the default. Codebases pinned to v2 for years; migration guides exist but the change touched every usage.

**`load()` returns a dict unless you make it not.** New users expect model instances back. Without a `@post_load` builder you get a plain dict, and forgetting this is a common source of `AttributeError` downstream.

**Type-checker blindness.** Because schemas don't project static types, IDE autocomplete and mypy can't see the loaded shape. Projects that want both static types and validation frequently choose pydantic or attrs+cattrs instead, or maintain a parallel `TypedDict`/dataclass by hand.

**Version and ecosystem pinning.** The core library and its satellites (marshmallow-sqlalchemy, flask-marshmallow, webargs, apispec) have historically needed coordinated version ranges; a major marshmallow bump can require simultaneous upgrades across the stack. marshmallow 4.x is the current line[^3]; confirm each companion library supports your major version before upgrading. The default branch is `dev`, not `main`/`master` — target PRs and changelog links accordingly.

## When to Use / When Not

**Use when:**
- You want serialization/validation decoupled from your ORM/domain models.
- One object needs several representations (public vs internal vs ingest).
- You're already on Flask + webargs + apispec, or building OpenAPI specs via apispec.
- You need all validation errors at once, as a structured nested dict.

**Avoid when:**
- Raw throughput matters — pydantic v2's Rust core is far faster.
- You want static types and editor/mypy awareness of validated shapes.
- You're on FastAPI, where pydantic is the native, zero-friction choice.
- You want model-first ergonomics (fields living on the domain class itself).

## Alternatives

- pydantic/pydantic — use instead when you want Rust-speed validation, static types, and FastAPI-native models.
- python-attrs/attrs + python-attrs/cattrs — use when you want lightweight classes plus explicit structure/unstructure without a schema DSL.
- lidatong/dataclasses-json — use when your data already lives in dataclasses and you want simple JSON round-tripping.
- encode/django-rest-framework — use its serializers instead when you're inside Django and want tight ORM/view integration.
- python-jsonschema/jsonschema — use when you need standards-compliant JSON Schema validation rather than a Python object mapper.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-11 | Initial release by Steven Loria[^1]. |
| 2.0 | 2015-09 | Stable field/validation API; `(data, errors)` tuple returns. |
| 3.0 | 2019-08 | Major rewrite: `load`/`dump` return data and raise `ValidationError`; `strict` removed; Python 2 dropped[^4]. |
| 4.0 | 2025 | Current major line; further API cleanup and legacy removals[^3]. |

## References

[^1]: marshmallow — project author Steven Loria (sloria); repository created 2013-11-10. https://github.com/marshmallow-code/marshmallow
[^2]: pydantic v2 rewrote its validation core in Rust (pydantic-core). https://docs.pydantic.dev/latest/
[^3]: marshmallow changelog and upgrading guide. https://marshmallow.readthedocs.io/en/latest/changelog.html
[^4]: marshmallow 3.0 upgrading guide (return-value and `ValidationError` changes). https://marshmallow.readthedocs.io/en/stable/upgrading.html

## Tags

python, serialization, deserialization, validation, schema, serde, orm-agnostic, rest-api, data-validation, marshalling
