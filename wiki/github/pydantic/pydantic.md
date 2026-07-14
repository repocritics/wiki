# pydantic/pydantic

> Data validation for Python driven by standard type hints, with a Rust core for speed.

[GitHub repo](https://github.com/pydantic/pydantic) ·
[Official website](https://pydantic.dev/docs/validation) ·
[License: MIT](https://github.com/pydantic/pydantic/blob/main/LICENSE)

## Overview

Pydantic is a data validation and parsing library for Python: you declare the shape of your data with ordinary type annotations, and Pydantic coerces, validates, and serializes runtime data against that schema[^1]. It was created by Samuel Colvin in 2017 and is now one of the most-depended-on packages in the Python ecosystem, pulled in transitively by FastAPI, LangChain, the AWS/OpenAI SDKs, and much of the modern data/agent stack. At ~28k stars the star count understates its reach; download volume (hundreds of millions/month) is the truer measure of how load-bearing it is.

The defining event in Pydantic's history is the **V1 → V2 rewrite**[^2]. V1 (through 1.10) was pure Python. V2 (June 2023) moved the validation engine into `pydantic-core`, a separate Rust crate compiled via PyO3, with the Python `pydantic` package becoming a schema-building and ergonomics layer on top. V2 is 5–50× faster depending on workload, but it renamed or removed a large fraction of the public API. That break still defines the library's day-to-day reality: a decade of tutorials, Stack Overflow answers, and LLM training data describe V1 APIs that no longer exist.

The central tension is **coercion vs. strictness**. By default Pydantic is lax: it will turn the string `"123"` into `int` `123`, `"2017-06-01"` into a `datetime`, and so on. This is convenient for parsing untrusted input (HTTP bodies, config files) but surprising for people who expect a type checker's exactness. V2 added a first-class `strict` mode to opt out per-field, per-model, or per-call; choosing where to sit on that spectrum is the main design decision when adopting it.

## Getting Started

```bash
pip install -U pydantic
# or: conda install pydantic -c conda-forge
```

```python
from datetime import datetime
from pydantic import BaseModel, EmailStr, ValidationError

class User(BaseModel):
    id: int
    name: str = "John Doe"
    signup_ts: datetime | None = None
    friends: list[int] = []

# Lax coercion: str/bytes become int, str becomes datetime
external = {"id": "123", "signup_ts": "2017-06-01 12:22", "friends": [1, "2", b"3"]}
user = User(**external)
print(user.id, user.friends)        #> 123 [1, 2, 3]

try:
    User(id="not-a-number")
except ValidationError as e:
    print(e.errors()[0]["type"])    #> int_parsing
```

For settings/env parsing, install the now-separate `pydantic-settings`; for `EmailStr` and similar, install `pydantic[email]`.

## Architecture / How It Works

Since V2, Pydantic is two packages that ship together:

1. **`pydantic-core`** — a Rust crate (PyO3 bindings) that holds the actual validators and serializers. It executes a "core schema": a JSON-like tree describing every field, validator, and default. All hot-path work (coercion, error collection, JSON parsing via the `jiter` crate, serialization) happens in compiled code.
2. **`pydantic`** (Python) — inspects your type hints and `Field()` declarations, builds the core schema once at class-definition time, and hands it to `pydantic-core`. This is where the ergonomic surface lives: `BaseModel`, validators, JSON Schema generation, config.

Because the core schema is built when the class is defined, model creation has a one-time cost and validation is cheap thereafter. `TypeAdapter` exposes the same machinery for arbitrary types (`list[int]`, `TypedDict`, dataclasses) without a `BaseModel` wrapper.

Key building blocks in V2:

- **Validators** — `@field_validator` (per field) and `@model_validator` (whole model), each with `mode="before"` (raw input) or `mode="after"` (typed value). The V1 `@validator`/`@root_validator` decorators still exist but are deprecated.
- **Serialization** — `model_dump()` / `model_dump_json()` replace V1's `.dict()` / `.json()`. Serialization is also driven by `pydantic-core`, with `@field_serializer` / `@computed_field` for customization.
- **`Annotated` metadata** — constraints attach via `typing.Annotated[int, Field(gt=0)]` or helpers like `conint`, keeping the type and its rules in one place.
- **JSON Schema** — models emit Draft 2020-12 JSON Schema, the mechanism FastAPI uses to build OpenAPI docs.
- **Discriminated unions** — `Field(discriminator="type")` selects a variant by a tag field, avoiding "try every member" union costs.

Config moved from an inner `class Config` to a `model_config = ConfigDict(...)` attribute. V2 ships the old V1 codebase under `from pydantic import v1`, so a project can import both during migration.

## Production Notes

**The V1→V2 migration is the dominant operational cost.** Renames are mechanical but pervasive: `.dict()`→`model_dump()`, `.json()`→`model_dump_json()`, `.parse_obj()`→`model_validate()`, `@validator`→`@field_validator`, `class Config`→`model_config`, `__fields__`→`model_fields`. Behavior changes are the real hazard — e.g. V2 no longer copies model instances on validation by default, `Optional[x]` no longer implies a default of `None`, and some coercion edge cases differ. `bump-pydantic` automates the easy renames; the semantic differences still need review.

**Pinning matters.** Because V1 and V2 coexist in the wild, transitive dependency conflicts are common: a library that hasn't migrated pins `pydantic<2`, another needs `>=2`. Check that your whole tree agrees before upgrading. Libraries commonly support both via the `pydantic.version.VERSION` check or the built-in `pydantic.v1` shim.

**Wheels and native builds.** `pydantic-core` is Rust, distributed as prebuilt wheels for common platforms. On unusual targets (some ARM/musl/Alpine images, exotic CPython builds) pip may fall back to compiling from source, which needs a Rust toolchain. Pin to images with manylinux/musllinux wheels available, or install `rust` in the build stage.

**Performance is real but not free.** V2 validation is fast, but building many distinct model classes at import time (large codegen'd schemas, deeply nested generics) adds startup latency and memory. `TypeAdapter` instances should be created once and reused, not per-call. `model_dump()` with `mode="json"` is heavier than `mode="python"`; for hot serialization paths prefer `model_dump_json()` which stays in Rust end to end.

**Strictness footguns.** Default lax coercion will silently accept `"1"` as `True`-adjacent ints, truncate/parse dates from strings, and coerce numeric strings. For API boundaries where you want exactness, set `model_config = ConfigDict(strict=True)` or use `Strict` annotations — otherwise malformed-but-coercible input passes validation.

**Mypy/IDE integration.** The `pydantic.mypy` plugin is recommended for accurate typing of generated `__init__` signatures and validators; without it, some patterns type-check poorly. V2's typing is materially better than V1's but still leans on the plugin for full fidelity.

## When to Use / When Not

**Use when:**
- You parse untrusted or external data (HTTP, config, message queues) and want typed, validated objects with clear errors.
- You're on FastAPI/LangChain/an LLM-agent stack where Pydantic is already the schema lingua franca.
- You need JSON Schema / OpenAPI generation from your models for free.
- You want structured LLM output validation — model schemas map cleanly to function/tool calling.

**Avoid when:**
- You only need static type checking with zero runtime cost — use plain dataclasses or `TypedDict`; Pydantic's validation overhead is wasted.
- You want an ORM — Pydantic validates shapes, it is not a database layer (pair it with SQLModel/SQLAlchemy instead).
- You're in a no-native-code or offline-build environment where shipping a Rust wheel is a problem, and V1's pure-Python path isn't an option.
- Ultra-hot inner loops over trusted data where even microseconds of per-object cost matter — validate at the boundary, then use plain objects inside.

## Alternatives

- attrs/attrs — class-boilerplate + optional validators, lighter and validation-optional; use when you want structured classes without a validation engine.
- python-attrs/cattrs — structuring/unstructuring on top of attrs; use when you want explicit converter control rather than annotation-driven coercion.
- lidatong/dataclasses-json — JSON (de)serialization for stdlib dataclasses; use when you just need dataclass ⇄ JSON and no coercion rules.
- keleshev/schema or python-jsonschema/jsonschema — dict/JSON-Schema validation; use when validating raw JSON documents rather than building typed models.
- msgspec (jcrist/msgspec) — very fast serialization + validation for msgpack/JSON; use when raw throughput beats Pydantic's ecosystem gravity.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017-06 | Initial release; pure-Python validation from type hints[^1]. |
| 1.0 | 2019-10 | First stable major; the API a generation of tutorials describe. |
| 1.10 | 2022-08 | Final V1 line; still maintained on the `1.10.X-fixes` branch. |
| 2.0 | 2023-06 | Ground-up rewrite; validation moves to Rust `pydantic-core`[^2]. |
| 2.x | 2023–2026 | Steady V2 line; `TypeAdapter`, strict mode, JSON Schema 2020-12, perf work. |

## References

[^1]: Pydantic documentation — "Get Started". https://pydantic.dev/docs/validation/latest/get-started/
[^2]: Samuel Colvin, "Pydantic V2 is here!" — 2023-06-30. https://pydantic.dev/articles/pydantic-v2-final

## Tags

python, data-validation, type-hints, serialization, json-schema, parsing, rust-core, pydantic, settings, fastapi
