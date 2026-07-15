# litestar-org/litestar

> A community-governed ASGI framework for building APIs in Python, built on msgspec and a layered configuration model.

[GitHub repo](https://github.com/litestar-org/litestar) ·
[Official website](https://docs.litestar.dev/) ·
[License: MIT](https://github.com/litestar-org/litestar/blob/main/LICENSE)

## Overview

Litestar is an ASGI web framework aimed squarely at API work: data validation, serialization, dependency injection, OpenAPI generation, and authorization primitives, with server-side rendering treated as a secondary concern. It occupies the same niche as FastAPI — typed Python handlers that produce OpenAPI schemas automatically — but arrives at it from a different set of engineering choices and a different governance model.

The project began life as **Starlite**, built on top of Starlette and Pydantic. It was renamed to Litestar with the 2.0 release in late 2023, and that release also rewrote the internals: Starlette is no longer a hard dependency, and [`msgspec`](https://github.com/jcrist/msgspec) became the default engine for validation and (de)serialization[^1]. The `starlite` PyPI package and import path are legacy; current code imports from `litestar`. A large fraction of tutorials, blog posts, and StackOverflow answers still say "Starlite," which is the single biggest source of confusion for newcomers.

The defining tension is ecosystem size versus design. Litestar is, in several respects, more coherent than FastAPI — a real DI container, a layered config system, first-class DTOs, pluggable serialization — but it is a community project without a single dominant corporate backer, and its third-party ecosystem (integrations, StackOverflow coverage, hiring pool) is a fraction of FastAPI's. As of 2026 it sits around 8.3k stars against FastAPI's tens of thousands.

## Getting Started

```shell
pip install litestar
# or with the CLI + uvicorn server:
pip install 'litestar[standard]'
```

```python
# app.py
from litestar import Litestar, get

@get("/")
async def hello_world() -> dict[str, str]:
    return {"hello": "world"}

app = Litestar(route_handlers=[hello_world])
```

```bash
litestar run   # dev server; add --reload for autoreload
```

The return type annotation is not optional decoration — Litestar reads it to build the OpenAPI schema and to serialize the response. Omitting the return type on a handler raises at startup rather than silently defaulting.

## Architecture / How It Works

**Layered configuration.** The central idea is that most settings — dependencies, guards, middleware, parameters, exception handlers, response config — can be declared at four levels: the `Litestar` app, `Router`, `Controller`, and individual handler. Lower levels override higher ones. Class-based `Controller`s group related routes under a shared `path` and shared config. This is more structured than FastAPI's router nesting, but it means that to know how a request is actually handled you sometimes have to trace resolution across all four layers.

**msgspec core.** Validation and serialization run through msgspec, a C-accelerated library that is meaningfully faster than pure-Python Pydantic v1 and competitive with Pydantic v2's Rust core. Litestar can consume `dataclasses`, `TypedDict`, msgspec `Struct`s, attrs classes, and Pydantic v1 *and* v2 models — including both Pydantic majors in the same application[^2]. Pydantic support is a compatibility layer over the msgspec path, not the native path.

**DTOs.** Data Transfer Objects are a first-class abstraction for reshaping data between the wire and your domain models — excluding fields, renaming, partial updates (`PATCH`), read/write asymmetry. `DTOData` gives handlers a validated-but-not-yet-instantiated payload. This is more built-out than FastAPI's `response_model`, and it is the main mechanism for exposing SQLAlchemy models without writing parallel schema classes.

**Dependency injection.** The DI system is explicitly modeled on pytest fixtures: named providers (`Provide(...)`) declared at any layer, resolved by parameter name, with sync/async support and per-request or app-scoped caching. It is not API-compatible with FastAPI's `Depends`, so migrations are not mechanical.

**Plugins.** Serialization, OpenAPI, DI, and app initialization are all extensible via the plugin protocol. SQLAlchemy support is the flagship plugin, developed as a separate package, [Advanced Alchemy](https://github.com/litestar-org/advanced-alchemy), which also ships repository and service abstractions.

OpenAPI output is 3.1.0, and the docs UI is pluggable across Scalar, Swagger-UI, ReDoc, RapiDoc, and Stoplight Elements. Trio is supported through AnyIO.

## Production Notes

**The Starlite rename is a live tax.** Search results, older Docker base images, and AI coding assistants trained on pre-2024 data routinely emit `from starlite import ...`. Pin to `litestar>=2` and treat any `starlite` reference as outdated.

**Ecosystem gaps.** There is no equivalent to the deep bench of FastAPI tutorials and integrations. Auth, background tasks, and admin tooling often mean reading the (good, but dense) official docs rather than copying a blog post. Hiring or onboarding developers who already know Litestar is unlikely; budget for ramp-up.

**Mixing type systems has a cost.** Running Pydantic v2 models everywhere is fine, but the fast path is msgspec `Struct`s. If serialization throughput matters, benchmark your actual models — the framework being "fast" does not guarantee your Pydantic-heavy handlers are.

**Layered config can obscure behavior.** A guard or dependency failing "somewhere" requires knowing which layer injected it. Teams that lean heavily on app-level defaults plus controller overrides should document the resolution order for their own sanity.

**Advanced Alchemy is a separate release cadence.** The SQLAlchemy integration lives in its own package and versions independently; upgrading Litestar and Advanced Alchemy together requires checking compatibility rather than assuming lockstep.

**2.x has been stable but iterative.** Minor releases within 2.x have introduced deprecations and occasional signature changes; read release notes before bumping. There is no LTS track — the supported version is the latest.

## When to Use / When Not

**Use when:**
- You want a typed API framework with a real DI container, DTOs, and layered config out of the box.
- You value serialization performance and are willing to use msgspec structs for hot paths.
- You need Pydantic v1 and v2 to coexist during a migration.
- You want OpenAPI 3.1 and multiple docs UIs without extra wiring.

**Avoid when:**
- You need the largest possible ecosystem, tutorial coverage, and hiring pool — FastAPI dominates there.
- Your team will be confused by Starlite-era material and you can't afford the ramp.
- You want a corporate-backed framework with a single accountable vendor.
- Your app is primarily server-rendered HTML rather than JSON APIs — Django or a template-first stack fits better.

## Alternatives

- tiangolo/fastapi — the incumbent; far larger ecosystem and mindshare, less structured DI and no first-class DTO layer. Use instead when ecosystem and hiring matter more than framework design.
- encode/starlette — the low-level ASGI toolkit Starlite was originally built on; use when you want minimal abstraction and will assemble validation/DI yourself.
- Neoteroi/BlackSheep — another fast, typed ASGI framework with DI; use when you want a similar feature set with a different API surface.
- sanic-org/sanic — performance-focused ASGI/async framework without the typed-schema focus; use when raw throughput and a mature ecosystem outweigh automatic OpenAPI.
- encode/django-rest-framework — batteries-included API layer on Django; use when you want the Django ORM/admin/auth ecosystem alongside your API.

## History

| Version | Date | Notes |
|---------|------|-------|
| Starlite 1.0 | 2022 | Initial stable release, built on Starlette + Pydantic[^3]. |
| Litestar 2.0 | 2023-10 | Rename from Starlite; Starlette dependency dropped; msgspec as core (de)serializer; Pydantic v1 + v2 support[^1]. |
| 2.x | 2024–2026 | Iterative minor releases; Advanced Alchemy split out; expanded docs UIs and plugin API. |

## References

[^1]: Litestar 2.0 release / "Starlite is now Litestar" announcement. https://litestar.dev/ · https://github.com/litestar-org/litestar/releases
[^2]: README, "Data Parsing, Type Hints, and Msgspec" — support for dataclasses, TypedDict, msgspec, Pydantic v1 and v2, and attrs. https://github.com/litestar-org/litestar
[^3]: Litestar (Starlite) documentation and migration guide. https://docs.litestar.dev/

## Tags

python, asgi, web-framework, rest-api, openapi, msgspec, pydantic, dependency-injection, async, backend
