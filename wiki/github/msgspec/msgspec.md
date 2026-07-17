# msgspec/msgspec

> A C-accelerated serialization and validation library for Python — JSON/MessagePack at near-C speed, with type-annotation-driven validation folded into the decode loop.

[GitHub repo](https://github.com/msgspec/msgspec) ·
[Official website](https://msgspec.dev) ·
[License: BSD-3-Clause](https://github.com/msgspec/msgspec/blob/main/LICENSE)

## Overview

msgspec is a Python library by Jim Crist-Harif that pairs a fast serialization layer with schema validation, covering JSON, MessagePack, YAML, and TOML[^1]. Its central claim is that validation need not cost anything: instead of decoding to generic `dict`/`list` and then walking the result to check types (the two-pass model of most validators), msgspec's C decoders consume the target type annotations and validate as they parse. The README's benchmark line — that it decodes *and* validates JSON faster than orjson decodes alone — is the whole thesis in one sentence[^1].

The project began in January 2021[^2] and predates pydantic's Rust-based v2 rewrite; for the decode-and-validate path it has historically stayed competitive with or ahead of pydantic-core. At roughly 3,900 stars and 157 forks it is an order of magnitude smaller than pydantic in mindshare, and it is deliberately narrower: no plugin system, no rich validator DSL, no ORM integration. It is a library for people who have a wire format and a set of typed schemas and want the conversion between them to be fast and correct.

The defining tradeoff is speed-and-strictness versus flexibility. msgspec gives you a small, closed set of well-optimized types and coercion rules; it does not give you pydantic's arbitrary `@field_validator` hooks or its ecosystem of plugins. When your validation needs fit its model, it is hard to beat. When they don't, you reach for `dec_hook`/`enc_hook` escape valves or a different library.

## Getting Started

```bash
pip install msgspec
# optional format extras:
pip install "msgspec[yaml]"   # pulls in PyYAML
pip install "msgspec[toml]"   # pulls in tomli/tomli_w on older Pythons
```

```python
import msgspec

class User(msgspec.Struct):
    name: str
    groups: set[str] = set()
    email: str | None = None

alice = User("alice", groups={"admin", "engineering"})

# encode -> bytes
buf = msgspec.json.encode(alice)
# b'{"name":"alice","groups":["admin","engineering"],"email":null}'

# decode + validate against the schema
msgspec.json.decode(buf, type=User)          # -> User(...)
msgspec.json.decode(b'{"name":"bob","groups":[123]}', type=User)
# msgspec.ValidationError: Expected `str`, got `int` - at `$.groups[0]`
```

The error path (`$.groups[0]`) is generated during decoding, not reconstructed afterward.

## Architecture / How It Works

msgspec is a CPython C extension. The performance comes from three coupled decisions:

1. **`msgspec.Struct` is a C-level type.** Structs behave like dataclasses/attrs — typed fields, defaults, `__init__`, `__repr__`, equality — but the layout and access paths are implemented in C rather than generated Python, which is where the "5–60x faster for common operations" figure originates[^1]. Structs support frozen instances, `gc=False` to opt out of cyclic GC tracking, keyword-only fields, and tagged unions.

2. **Validation is fused into the decoder.** `decode(buf, type=T)` walks the bytes and the type `T` simultaneously. There is no intermediate generic object graph for typed decodes, so the "validation pass" that other libraries pay for does not exist as a separate step. This is the source of the zero-cost claim and also of its limitations: the validator only understands the type shapes it has C code for.

3. **A fixed but broad type vocabulary.** Beyond Structs, msgspec validates standard `dataclasses`, `attrs` classes, `TypedDict`, `NamedTuple`, `Enum`/`IntEnum`, `datetime`/`date`/`time`, `UUID`, `Decimal`, `bytes`, and the usual container generics. Constraints (min/max, regex, length) are expressed with `msgspec.Meta` inside `typing.Annotated`. Custom types are handled through `dec_hook`/`enc_hook` callbacks rather than per-field validators.

The four supported formats are not equal citizens. **JSON and MessagePack are native C encoders/decoders** and are the fast paths. **YAML and TOML are thin wrappers**: `msgspec.yaml` delegates parsing to PyYAML and `msgspec.toml` to the standard-library `tomllib` (or `tomli`/`tomli_w`), then routes the resulting Python objects through msgspec's `convert()` machinery for typed validation. So YAML/TOML get msgspec's validation but not its raw parsing speed — they inherit the underlying library's performance.

Polymorphic decoding is handled by **tagged unions**: annotate Structs with a `tag`/`tag_field` and a `Union[A, B, C]` decode dispatches on the discriminator without try-each-branch guessing.

## Production Notes

- **`strict=False` changes the coercion rules.** In strict mode (default) `"123"` will not satisfy an `int` field; in lax mode msgspec coerces strings to numbers/bools where unambiguous. Lax mode is useful for query strings and environment-style config, but flipping it silently widens what your schema accepts — set it deliberately per call site.

- **Struct defaults and the mutable-default question.** msgspec handles field defaults at the C level and supports `field(default_factory=...)` semantics, so a `list`/`dict`/`set` default does not produce the classic shared-mutable-default bug the way a naive dataclass would. Still, know whether a field is frozen; `frozen=True` Structs are hashable and safe to share, plain ones are not.

- **It's a compiled extension.** msgspec ships binary wheels for mainstream platforms, but on niche architectures or restricted build environments you pay for a C compile, and there is no pure-Python fallback. Factor this into Alpine/musl and exotic-arch deployments.

- **No validator ecosystem.** There is no equivalent of pydantic's `@field_validator`, computed fields, or the pydantic plugin surface. Cross-field or business-rule validation lives in your own code after decode, or in `dec_hook`. Teams migrating from pydantic for the speed often rediscover that they were leaning on those hooks.

- **Optional deps are real deps for YAML/TOML.** `import msgspec` works with zero dependencies, but `msgspec.yaml` needs PyYAML installed and TOML encoding needs `tomli_w` on Pythons without `tomllib` writers. Missing extras fail at call time, not import time.

- **Schema evolution.** Unknown fields are ignored by default on decode, which is forgiving for forward-compatible messages; use Structs with explicit fields plus tagged unions rather than free-form dicts if you want strictness. msgspec can also emit JSON Schema from your types (`msgspec.json.schema`), which is worth wiring into API tooling.

## When to Use / When Not

**Use when:**
- You have a defined wire format (JSON/MessagePack especially) and typed schemas, and decode/encode throughput matters.
- You want validation without a heavy dependency or a large import-time cost.
- You're building high-QPS services, message consumers, or data pipelines where the two-pass decode-then-validate overhead shows up in profiles.
- You want JSON Schema generation from Python types for free.

**Avoid when:**
- Your validation is rule-heavy (custom validators, cross-field logic, computed serializers) — pydantic's model is a better fit even if slower.
- You need a pure-Python library with no compiled extension.
- YAML/TOML are your hot path — you'll be bound by PyYAML/tomllib, not by msgspec.
- You want a large plugin ecosystem and framework integrations out of the box.

## Alternatives

- pydantic/pydantic — use instead when you need rich validators, computed fields, settings management, or framework integration; broader ecosystem, generally slower on pure decode+validate.
- ijl/orjson — use instead when you only need very fast JSON encode/decode and no schema validation at all.
- python-attrs/cattrs — use instead when your models are attrs/dataclasses and you want structuring/unstructuring in pure Python without a wire-format focus.
- msgpack/msgpack-python — use instead when you want MessagePack alone without validation or the Struct type.
- marshmallow-code/marshmallow — use instead when you want an established, pure-Python schema/serialization library with explicit field objects.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021 | Initial release: `Struct` type + native JSON/MessagePack encoders[^2]. |
| 0.x | 2022 | Typed decode/validation matured; constraints via `Meta` + `Annotated`; tagged unions. |
| 0.18.x | 2023 | YAML and TOML support added (validation over PyYAML / `tomllib`). |
| 0.19.x | 2024 | Current line; expanded standard-library type coverage and newer-Python support. |

## References

[^1]: msgspec README and documentation. https://msgspec.dev
[^2]: Repository created 2021-01-26 (GitHub API `repos/msgspec/msgspec`). https://github.com/msgspec/msgspec

## Tags

python, serialization, validation, json, messagepack, toml, yaml, c-extension, schema, performance, deserialization, json-schema
