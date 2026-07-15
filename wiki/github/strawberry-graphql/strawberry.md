# strawberry-graphql/strawberry

> A code-first GraphQL library for Python that derives the schema from type annotations and dataclass-style decorators.

[GitHub repo](https://github.com/strawberry-graphql/strawberry) ·
[Official website](https://strawberry.rocks) ·
[License: MIT](https://github.com/strawberry-graphql/strawberry/blob/main/LICENSE)

## Overview

Strawberry is a Python GraphQL library, created by Patrick Arminio in 2018[^1], built around the idea that a GraphQL schema should fall out of ordinary Python type hints. You annotate a class with `@strawberry.type`, declare fields as typed attributes or methods, and the library reflects those annotations into a GraphQL schema at import time. This is the "code-first" model: Python is the source of truth, the SDL is generated. It contrasts with schema-first libraries (Ariadne, Tartiflette) where you author GraphQL SDL by hand and bind resolvers to it.

Strawberry does not implement GraphQL execution itself. It sits on top of `graphql-core`[^2] — the Python port of `graphql-js` — and uses it for parsing, validation, and execution, while owning the type-definition ergonomics, integrations, and extension layer. The practical consequence is that Strawberry's correctness and raw execution speed are largely inherited from `graphql-core`, and its value-add is the developer experience: async-first resolvers, editor autocompletion, and a `mypy` plugin so the schema types are also the types your IDE checks.

The defining tension is versioning discipline versus iteration speed. Strawberry has stayed on a perpetual `0.x` line and ships releases very frequently — every merged PR carries a `RELEASE.md` note that bumps the version automatically. That keeps the project moving but means there is no semantic-versioning contract; behavior can shift across minor bumps, so pinning is not optional.

## Getting Started

```shell
pip install "strawberry-graphql[cli]"
```

```python
# app.py
import strawberry


@strawberry.type
class User:
    name: str
    age: int


@strawberry.type
class Query:
    @strawberry.field
    def user(self) -> User:
        return User(name="Patrick", age=100)


schema = strawberry.Schema(query=Query)
```

```shell
strawberry dev app   # serves GraphiQL at http://0.0.0.0:8000/graphql
```

The `Query` type's return annotations (`-> User`) are what produce the schema; there is no separate SDL file to keep in sync. For production you mount `schema` under an ASGI/WSGI integration rather than the dev server.

## Architecture / How It Works

The core is a reflection layer. Decorators (`@strawberry.type`, `@strawberry.input`, `@strawberry.enum`, `@strawberry.interface`, `@strawberry.federation.type`) attach metadata to classes; `strawberry.Schema(...)` walks the annotated graph and constructs the corresponding `graphql-core` `GraphQLSchema`. Because this happens at import time from real Python types, forward references and circular type relationships rely on string annotations and `typing` lazy evaluation — a common source of import-order errors in large schemas.

Field resolution is async-first. Resolvers may be sync or `async def`; Strawberry adapts both. Cross-cutting behavior is handled by **extensions** (schema extensions and field extensions) — a middleware chain around parsing, validation, and execution used for tracing, caching, masking, and query-cost limiting. **Permissions** are implemented as field extensions (`permission_classes`).

Integrations are first-class and numerous: ASGI (Starlette), FastAPI, Django (sync and async views, plus a separate `strawberry-graphql-django` project for ORM integration), Flask, Channels, aiohttp, Sanic, Litestar, and Chalice. Subscriptions are supported over WebSockets (`graphql-transport-ws` and the legacy `graphql-ws` protocols) and, more recently, over Server-Sent Events. Apollo **Federation** (v1 and v2) is supported directly through `strawberry.federation`.

The `mypy` plugin and Pyright support exist because the decorator-heavy API is otherwise opaque to static type checkers — without the plugin, `mypy` cannot see that `@strawberry.type` preserves the dataclass-like constructor signature. This tight coupling to the type checker is deliberate: the schema and the type-checked code are the same artifact.

## Production Notes

- **Execution speed is `graphql-core`'s speed.** Strawberry adds a thin reflection layer, but query execution, validation, and parsing run in pure-Python `graphql-core`. Large, deeply-nested queries over big schemas are CPU-bound and will not match a compiled/Node GraphQL server. Profile before assuming the bottleneck is your resolvers.
- **N+1 is on you.** Nothing batches database access automatically. Strawberry ships a `DataLoader` implementation, but wiring loaders per-request (usually via context) is manual and easy to forget, producing per-field query storms under nested selections.
- **No default query-cost protection.** Introspection is on by default and there is no built-in depth or complexity limit. A public endpoint should add a query-depth/cost extension and disable introspection in production, or it is trivially DoS-able with a recursive query.
- **Version pinning is mandatory.** The `0.x` continuous-release model means minor bumps can carry behavioral or API changes. Pin an exact version (or a narrow range) and read `RELEASE.md`/changelog entries before upgrading; "just bump it" is not safe here.
- **Import-time schema construction.** Schema build happens when the module is imported, so type-resolution errors, missing forward references, and misconfigured generics surface as import failures, not request-time errors — annoying in dev, but it fails fast.
- **Type-checker coupling.** If you skip the `mypy` plugin (or use a checker it doesn't cover well), you lose much of the point of the library and will fight false positives on decorated classes.

## When to Use / When Not

**Use when:**
- You want the schema to be generated from Python type hints and checked by `mypy`/Pyright, with the schema and application code as a single typed artifact.
- Your stack is async (FastAPI, Starlette, ASGI Django) and you want async resolvers and subscriptions without adapters.
- You need Apollo Federation from a Python subgraph.

**Avoid when:**
- You prefer SDL as the contract and want designers/frontend to own the schema file — a schema-first library fits that workflow better.
- You need maximum execution throughput for very large schemas; any `graphql-core`-based server is Python-bound, and a non-Python GraphQL server may be the right call.
- You require strict semantic-versioning guarantees and infrequent, well-telegraphed breaking changes.

## Alternatives

- graphql-python/graphene — the older, most-established Python GraphQL library; code-first but class/`Field`-based rather than type-annotation-native. Use when you want the largest ecosystem and don't need annotation-driven typing.
- mirumee/ariadne — schema-first (SDL + resolver binding). Use when the GraphQL SDL should be the source of truth rather than Python types.
- graphql-python/graphql-core — the low-level execution engine Strawberry itself depends on. Use directly only when you want to build your own schema-construction layer.
- tartiflette/tartiflette — async, SDL-first engine. Consider when you want schema-first and async, though maintenance activity is lower.
- strawberry-graphql/strawberry-graphql-django — companion project (not an alternative) for Django ORM integration; reach for it when pairing Strawberry with Django models.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2018-12 | Project created by Patrick Arminio[^1]. |
| 0.x (ongoing) | 2019– | Continuous release line; version bumped per-PR via `RELEASE.md`[^3]. |
| — | — | Adopted as a recommended GraphQL option in FastAPI's documentation[^4]. |
| — | — | Apollo Federation v2 and SSE subscription support added over the 0.x line. |

Strawberry has intentionally never cut a `1.0`; treat the changelog, not the version number, as the migration signal.

## References

[^1]: Strawberry GraphQL repository and history — created 2018-12-21. https://github.com/strawberry-graphql/strawberry
[^2]: `graphql-core`, the Python port of `graphql-js` used as Strawberry's execution engine. https://github.com/graphql-python/graphql-core
[^3]: Contributing guide describing the per-PR `RELEASE.md` versioning workflow. https://github.com/strawberry-graphql/strawberry/blob/main/CONTRIBUTING.md
[^4]: FastAPI documentation, GraphQL section recommending Strawberry. https://fastapi.tiangolo.com/how-to/graphql/

## Tags

python, graphql, graphql-server, code-first, type-annotations, async, asgi, fastapi, django, federation, api, dataclasses
