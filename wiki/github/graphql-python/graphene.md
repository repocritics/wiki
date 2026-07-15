# graphql-python/graphene

> Code-first GraphQL schema library for Python — you write Python classes, it produces a GraphQL schema.

[GitHub repo](https://github.com/graphql-python/graphene) ·
[Official website](https://graphene-python.org) ·
[License: MIT](https://github.com/graphql-python/graphene/blob/master/LICENSE)

## Overview

Graphene is a Python library for defining GraphQL schemas in code rather than in
SDL. You declare types as Python classes (`graphene.ObjectType`, `graphene.Mutation`,
etc.) and Graphene builds the underlying schema and wires resolver methods to fields.
It has been the default answer to "how do I do GraphQL in Python" since the mid-2010s,
largely because of its Django integration[^1].

The library itself is deliberately data-agnostic: the core package knows nothing about
databases and only produces a schema plus an execution entry point. Almost all of its
real-world value comes from separate integration packages — `graphene-django`,
`graphene-sqlalchemy`, `graphene-mongo`, `graphene-federation` — each maintained on its
own cadence[^2]. Many teams never touch the core API directly; they use it through
`graphene-django`. This split is the defining tension: the base library is stable and
small, but the surface you actually depend on lives in adjacent repos with uneven
maintenance.

The second tension is momentum. The 8.2k-star repo remains the most-starred Python
GraphQL project, but its release cadence has slowed since the 3.x line landed and the last
push to `master` was in late 2025[^3]. Its code-first niche is now actively contested by
Strawberry, which offers the same "Python classes to schema" model with modern type hints
and async-first execution.

## Getting Started

```bash
pip install "graphene>=3.1"
```

```python
import graphene

class Query(graphene.ObjectType):
    hello = graphene.String(name=graphene.String(default_value="World"))

    def resolve_hello(self, info, name):
        return f"Hello {name}"

schema = graphene.Schema(query=Query)

result = schema.execute('{ hello(name: "Graphene") }')
print(result.data["hello"])   # "Hello Graphene"
```

Resolvers are `resolve_<field>` methods (or plain functions). The second positional
argument, `info`, carries the execution context and request-scoped state — the primary
channel for passing auth/session data into resolvers.

## Architecture / How It Works

Graphene is a DSL layer that sits on top of **graphql-core**, the Python port of
graphql-js that does the actual parsing, validation, and execution[^4]. Graphene's job
is to translate declarative Python classes into a `graphql-core` `GraphQLSchema`; at
runtime, execution is graphql-core's, not Graphene's.

Type definitions rely on a metaclass. When you subclass `graphene.ObjectType`, a metaclass
collects the class-level field attributes and registers them. Supported constructs mirror
the GraphQL spec: `ObjectType`, `InputObjectType`, `Enum`, `Interface`, `Union`, scalars,
and `Mutation` (a mutation is an `ObjectType` with a `mutate` classmethod and an
`Arguments` inner class).

The `graphene.relay` module ships Relay server conventions out of the box: the `Node`
interface with opaque global IDs, `Connection`/`Edge` cursor pagination types, and
`ClientIDMutation`. This is why "Graphene + Relay" was a common early pairing.

Execution is synchronous by default via `schema.execute(...)`; async uses
`schema.execute_async(...)` and async resolvers. Graphene 3 does **not** ship a
batching/`DataLoader` layer the way the old promise-based v2 did — the async `DataLoader`
pattern is added via `aiodataloader` or an integration package.

The coupling to watch is the core ↔ integration boundary. `graphene-django` maps Django
models and querysets to types and injects its own resolvers, connection fields, and a
`DjangoFilterConnectionField`; `graphene-sqlalchemy` does the analogous thing for SQLAlchemy.
These packages track — and sometimes lag — both Graphene and their ORM, so an upgrade of
Graphene or Django can be gated on whichever integration is slowest to follow.

## Production Notes

**N+1 queries are the default failure mode.** Resolvers fire per field per object, so a
list field that resolves a related record will issue one query per item unless you batch.
There is no turnkey batching in core; you add `DataLoader` (async) or push joins/prefetch
into the ORM layer (`select_related`/`prefetch_related` with `graphene-django`). Teams that
skip this discover it under load, not in tests.

**No built-in query depth or complexity limiting.** A public GraphQL endpoint with cyclic
types (very easy to create with Relay connections) is a denial-of-service vector: a deeply
nested query can fan out arbitrarily. You must add depth/complexity validation yourself
(custom validation rules on graphql-core) or via a third-party package. This is a common
security gap in Graphene deployments.

**Error masking.** By default, exceptions raised in resolvers surface as GraphQL errors,
and internal messages can leak to clients. Production setups need an error formatter /
middleware to mask unexpected exceptions and log the originals.

**The 2 → 3 migration was disruptive.** Graphene 3 moved from graphql-core 2 to 3, changed
resolver signatures and `info` handling, dropped several v2 conveniences, and required
coordinated upgrades of every integration package[^5]. Many projects pinned to Graphene 2
for years; treat a 2 → 3 jump as a real migration, not a version bump.

**Maintenance velocity.** Core is stable but slow-moving, and integration packages vary
widely in freshness. Before committing, check the specific integration you need
(`graphene-sqlalchemy`, `graphene-mongo`, etc.) for recent releases and Python/ORM version
support — the core repo's health does not guarantee the integration's.

**Performance.** Execution is pure-Python graphql-core — adequate for typical API workloads
but not a low-latency, high-throughput engine. If per-request GraphQL overhead is a
bottleneck, that is a library-level constraint, not a tuning problem.

## When to Use / When Not

**Use when:**
- You have a Django project and want GraphQL — `graphene-django` remains the most
  established path.
- You prefer defining schema in Python code over writing SDL.
- You need Relay server conventions (global IDs, connections) without hand-rolling them.
- You want a mature, MIT-licensed library with a large existing body of examples.

**Avoid when:**
- You're starting a new, async-heavy service and want active development — Strawberry is
  the more actively maintained code-first option.
- You prefer a schema-first (SDL) workflow — Ariadne fits that model directly.
- You need modern Python type-hint-driven definitions and editor inference as a first-class
  feature.
- Your integration package (non-Django ORM, exotic data source) shows signs of being stale.

## Alternatives

- strawberry-graphql/strawberry — code-first using Python type hints and dataclasses,
  async-first, actively maintained; use for new projects wanting Graphene's model with
  modern typing.
- mirumee/ariadne — schema-first (define SDL, bind resolvers); use when you want the schema
  as the source of truth rather than Python classes.
- tartiflette/tartiflette — async/SDL-first engine built for asyncio; use for asyncio-native
  services that prefer SDL.
- graphql-python/graphql-core — the execution engine Graphene sits on; use directly when you
  want low-level control and no DSL layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2015 | First public release; code-first schema DSL for Python[^1]. |
| 1.0 | 2016 | Stabilized core type system and resolver model. |
| 2.0 | 2018 | Major API changes; built on graphql-core 2. |
| 3.0 | 2022 | Rewrite onto graphql-core 3; breaking resolver/type changes[^5]. |
| 3.1+ | 2022+ | Current line; incremental fixes, slower cadence. |

## References

[^1]: Graphene documentation and project site. https://graphene-python.org
[^2]: Integration packages listed in the project README (graphene-django, graphene-sqlalchemy, graphene-mongo, graphene-federation). https://github.com/graphql-python/graphene
[^3]: Repository metadata via GitHub API: 8,235 stars, 819 forks, MIT, last push 2025-09-04 (retrieved 2026-07-15). https://github.com/graphql-python/graphene
[^4]: graphql-core — Python port of graphql-js, Graphene's execution engine. https://github.com/graphql-python/graphql-core
[^5]: Graphene v3 release notes / upgrade guide (graphql-core 3 migration). https://docs.graphene-python.org/en/latest/

## Tags

python, graphql, api, code-first, schema, relay, django, backend, web-framework, library
