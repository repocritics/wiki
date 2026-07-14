# fastapi/fastapi

> A Python web framework that derives request validation, serialization, and OpenAPI docs from standard type hints.

[GitHub repo](https://github.com/fastapi/fastapi) ·
[Official website](https://fastapi.tiangolo.com/) ·
[License: MIT](https://github.com/fastapi/fastapi/blob/master/LICENSE)

## Overview

FastAPI is an ASGI web framework created by Sebastián Ramírez (tiangolo), first
released in December 2018[^1]. Its central idea: you annotate function
parameters with Python type hints, and the framework uses those annotations to
validate incoming data, serialize responses, generate an OpenAPI schema, and
render interactive docs — with no separate schema definition. It occupies the
space between micro-frameworks (Flask) and batteries-included ones (Django),
targeting JSON APIs rather than server-rendered HTML.

FastAPI is not a monolith. It is a thin layer over two libraries it does not
own: **Starlette** provides the ASGI application, routing, requests/responses,
middleware, WebSockets, and background tasks; **Pydantic** provides the data
validation and serialization[^2]. FastAPI itself contributes the dependency
injection system, parameter declaration, security utilities, and OpenAPI
generation. This is the framework's defining tension: much of its behavior,
performance, and breaking changes originate upstream. The Pydantic v1→v2
migration (2023) is the clearest example — a rewrite of the validation core
that FastAPI users had to absorb.

As of 2026 the repository has ~100k stars and is one of the most-used Python API
frameworks in production, adopted at Microsoft, Uber, and Netflix among
others[^3]. The project moved from the personal `tiangolo/fastapi` namespace to
the `fastapi/` organization, and the author has since launched FastAPI Cloud, a
commercial hosting product that now funds the open-source work[^4].

## Getting Started

```bash
pip install "fastapi[standard]"   # quotes required in zsh; pulls uvicorn + CLI
```

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool | None = None

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}

@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item):
    return {"item_name": item.name, "item_id": item_id}
```

```bash
fastapi dev          # dev server with autoreload (wraps uvicorn)
fastapi run          # production mode, no reload
# interactive docs at /docs (Swagger UI) and /redoc (ReDoc)
```

The `item_id: int` hint produces path validation and coercion; the `Item` model
produces JSON-body validation and the OpenAPI schema, all from one declaration.

## Architecture / How It Works

The request lifecycle is worth understanding because it explains most footguns:

1. **Starlette** receives the ASGI request and routes it to the matched path
   operation. FastAPI's `APIRouter` and `FastAPI` app subclass Starlette's.
2. FastAPI inspects the endpoint signature once, at startup, building a
   dependency graph. Each parameter is classified by type and default into a
   source — path, query, header, cookie, body, form, or a `Depends(...)`.
3. Per request, it solves that graph: resolving dependencies (with caching
   within a request), validating each input through Pydantic, and calling the
   endpoint. The return value is validated/serialized against `response_model`.

**Dependency injection** is the framework's most distinctive feature.
`Depends(fn)` makes `fn`'s result an input; dependencies can themselves have
dependencies, can be `yield`-based (for setup/teardown, e.g. DB sessions), and
are cached per request. Security schemes (`OAuth2PasswordBearer`, API keys) are
implemented as dependencies, which is why auth "just plugs in" but is also why
there is no auth system — only primitives.

**Sync vs async is the biggest correctness trap.** A `def` endpoint (or `def`
dependency) is run in an external threadpool so it does not block the event
loop; an `async def` endpoint runs directly on the loop[^5]. Putting a blocking
call (a synchronous DB driver, `requests`, `time.sleep`) inside an `async def`
stalls the entire worker. The rule that trips people: if your handler is not
truly non-blocking, declare it `def`, not `async def`.

**OpenAPI** is generated lazily from the collected models and served at
`/openapi.json`; `/docs` and `/redoc` are just HTML shells pointed at it. This
is generation, not annotation — the schema reflects the code, which is the
selling point and also means schema surprises are code surprises.

## Production Notes

**FastAPI does not serve itself.** You run it under an ASGI server — Uvicorn is
standard, often behind Gunicorn (`gunicorn -k uvicorn.workers.UvicornWorker`) or
via `--workers` for multiple processes. There is no built-in worker manager for
production beyond what the CLI wraps.

**The async threadpool has a ceiling.** Sync (`def`) endpoints run in Starlette's
`AnyIO` threadpool, historically capped around 40 threads. An app that is mostly
sync handlers doing blocking I/O can saturate that pool and queue requests even
though the event loop is idle. Diagnosing this requires knowing your endpoints
are sync in the first place.

**Pydantic dominates the CPU profile.** For large or deeply nested response
models, validation and serialization — not your business logic — is frequently
the hot path. Pydantic v2's Rust core (`pydantic-core`) improved this
substantially over v1, but `response_model` still validates every response by
default. Dropping `response_model` or returning a plain `Response`/`ORJSONResponse`
is the usual escape for hot endpoints.

**Upgrade pain concentrates in dependencies, not FastAPI.** The Pydantic v1→v2
change altered validator syntax, config, and error shapes; projects pinned to v1
lived on `pydantic.v1` shims for a long time. Starlette version bumps
occasionally shift middleware or background-task behavior. Pin both.

**Background tasks are in-process.** `BackgroundTasks` runs after the response in
the same worker — fine for sending an email, wrong for anything that must
survive a restart or scale independently. Reach for Celery, Arq, or Dramatiq for
real job queues.

**No ORM, no migrations, no admin.** Unlike Django, FastAPI ships none of these.
SQLModel (also by tiangolo) or SQLAlchemy + Alembic are the common stack. This
is deliberate minimalism, but it means more assembly for CRUD-heavy apps.

## When to Use / When Not

**Use when:**
- You are building JSON/REST APIs and want validation plus OpenAPI docs for free.
- Your team already writes typed Python and wants editor autocomplete on request data.
- You need async I/O (external APIs, async DB drivers, WebSockets, streaming).
- You want automatic, client-generatable OpenAPI without hand-writing schemas.

**Avoid when:**
- You are building a server-rendered, template-heavy site with an admin — Django fits better.
- Your workload is entirely synchronous and blocking; the async model adds surface without benefit.
- You want one framework that bundles ORM, auth, migrations, and admin out of the box.
- You cannot tolerate depending on Starlette and Pydantic release cadences.

## Alternatives

- django/django — full-stack framework with ORM, admin, and auth; use it when you want batteries included and server-rendered pages, not just an API.
- pallets/flask — minimal WSGI micro-framework; use it when you want maximum control, a mature sync ecosystem, and don't need built-in validation or async.
- encode/starlette — the ASGI toolkit FastAPI is built on; use it directly when you want the async core without the Pydantic/DI/OpenAPI layer.
- litestar-org/litestar — async, typed, OpenAPI-first like FastAPI but with a batteries-included stance (built-in DI, ORM plugins); use it when you want FastAPI's ideas with more framework included.
- django/django + django-rest-framework — use when your API sits alongside an existing Django app and you want its serializers and permissions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-12-08 | Initial release[^1]. |
| 0.62 | 2020-11 | Pydantic-driven settings, expanded docs era. |
| 0.100 | 2023-07 | Pydantic v2 support (major validation-core change)[^2]. |
| 0.111 | 2024-05 | `fastapi` CLI (`fastapi dev` / `fastapi run`) introduced. |
| 0.115 | 2024-09 | `Query`/`Path`/etc. metadata and `Annotated` refinements. |
| 0.1x | 2026 | Actively maintained; no 1.0 tag — the 0.x line has been production-used for years despite the version number[^6]. |

## References

[^1]: FastAPI first commit and 0.1.0 release, December 2018. https://github.com/fastapi/fastapi/releases
[^2]: FastAPI announcement of Pydantic v2 support (0.100.0), July 2023. https://fastapi.tiangolo.com/release-notes/
[^3]: Adoption quotes (Microsoft, Uber/Ludwig, Netflix/Dispatch, Cisco) in the project README. https://github.com/fastapi/fastapi#opinions
[^4]: FastAPI Cloud — commercial hosting by the FastAPI team, primary sponsor of the OSS project. https://fastapicloud.com
[^5]: FastAPI docs, "Concurrency and async / await" — sync `def` runs in a threadpool, `async def` on the event loop. https://fastapi.tiangolo.com/async/
[^6]: FastAPI release notes / versioning — the project remains on 0.x. https://fastapi.tiangolo.com/release-notes/

## Tags

python, web-framework, rest-api, asgi, async, openapi, pydantic, starlette, type-hints, backend
