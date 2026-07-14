# Kludex/starlette

> The minimal ASGI toolkit that most other Python async frameworks — FastAPI above all — are built on top of.

[GitHub repo](https://github.com/Kludex/starlette) ·
[Official website](https://starlette.dev) ·
[License: BSD-3-Clause](https://github.com/Kludex/starlette/blob/main/LICENSE.md)

## Overview

Starlette is a small ASGI framework/toolkit for building async HTTP and WebSocket services in Python. It was created by Tom Christie under the `encode` organization in 2018 as the async successor to the WSGI-era ecosystem, and deliberately stops at the transport layer: routing, requests/responses, middleware, WebSockets, background tasks, and a lifespan protocol. It ships no ORM, no validation, no data layer, and no bundled server. Maintenance has since moved to Marcelo Trylesinski (`Kludex`), and the canonical repo is now `Kludex/starlette`; `encode/starlette` redirects there.[^1]

The reason Starlette matters out of proportion to its size is that FastAPI is built directly on it — every FastAPI request is a Starlette request, every FastAPI route is a Starlette route.[^2] That makes Starlette one of the most-deployed pieces of Python web infrastructure, usually without the operator knowing they depend on it. Understanding Starlette is the fastest way to understand what FastAPI does and does not do for you.

The defining tension is scope. Starlette's whole thesis is "few hard dependencies, do one layer well, compose the rest." That keeps it fast and legible, but means anything resembling a real application — validation, auth, serialization, DB sessions — is your job or a higher framework's. Teams that reach for Starlette directly are usually building a service thin enough that FastAPI's machinery would be overhead, or building their own framework on top.

## Getting Started

```shell
pip install starlette
pip install uvicorn        # Starlette does not ship a server; you supply an ASGI one
```

```python
# main.py
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route


async def homepage(request):
    return JSONResponse({"hello": "world"})


app = Starlette(routes=[Route("/", homepage)])
```

```shell
uvicorn main:app --reload
```

Optional extras are pulled in only when used: `python-multipart` for form parsing, `jinja2` for templates, `itsdangerous` for signed sessions, `httpx` for the `TestClient`. `pip install starlette[full]` installs the lot.

## Architecture / How It Works

Starlette is, at core, the ASGI application signature made ergonomic. Everything reduces to `async def app(scope, receive, send)`. A bare response is itself a callable that speaks that protocol:

```python
async def app(scope, receive, send):
    response = PlainTextResponse("Hello, world!")
    await response(scope, receive, send)
```

The `Starlette` application object composes three layers around that core:

1. **Routing** — `Route`, `WebSocketRoute`, `Mount`, and `Host` map incoming `scope` paths to endpoints. `Mount` lets you graft an entire sub-application (any ASGI app) under a path prefix, which is how larger apps are decomposed.
2. **Middleware stack** — every middleware is itself an ASGI app that wraps the next one. Middleware is applied in reverse of registration order: the last one added is the outermost. This is the single most common source of "why did my middleware not see X" confusion.
3. **Lifespan** — startup/shutdown handled through the ASGI lifespan protocol, exposed as `on_startup`/`on_shutdown` callbacks or the newer `lifespan` async context manager. This is where DB pools and background workers are opened and closed.

Concurrency is abstracted through **anyio**, so Starlette runs on both `asyncio` and `trio`, and synchronous endpoints are automatically run in a worker threadpool rather than blocking the event loop.[^3] Requests expose lazy, streaming access to the body (`await request.body()`, `request.stream()`, `request.form()`); responses include streaming and file variants. The `TestClient` is built on `httpx` and drives the app in-process without a live server.

Because each piece is an independent ASGI-compatible unit, Starlette is genuinely usable as a toolkit — you can import just its responses, or just its routing, into another ASGI framework. FastAPI exploits exactly this: it subclasses the application, reuses the routing and middleware, and layers Pydantic-based validation and OpenAPI generation on top.

## Production Notes

**You must bring a server.** Starlette does not serve HTTP. In production it runs behind uvicorn, hypercorn, or daphne, typically with a process manager and a reverse proxy (nginx/Caddy) or a cloud ASGI runtime. `StaticFiles` exists but is for development convenience; serve real static assets from a CDN or the proxy.

**`BaseHTTPMiddleware` is the classic footgun.** The convenient `BaseHTTPMiddleware` class historically buffered responses, broke streaming/background tasks in some configurations, and did not propagate `contextvars` set inside it to the endpoint.[^4] For anything performance- or streaming-sensitive, write a pure ASGI middleware (a plain `async def` wrapper) instead. Much of the ecosystem's middleware advice reduces to "prefer raw ASGI middleware."

**Async discipline.** An `async def` endpoint that makes a blocking call (a sync DB driver, `requests`, heavy CPU work) stalls the whole event loop for every connection on that worker. Either use async-native clients, or declare the endpoint `def` so Starlette offloads it to the threadpool. The threadpool has a bounded size, so it is a mitigation, not a license to block.

**Sessions are signed cookies, not server state.** `SessionMiddleware` stores the session in an `itsdangerous`-signed cookie: tamper-evident but not encrypted, size-limited (~4KB), and fully client-side. Do not put secrets or large payloads there; use a server-side store for real session state.

**Versioning is 0.x and not always additive.** Starlette has spent its whole life on `0.x` releases, and minor bumps have occasionally carried behavior or API changes. Downstream frameworks (FastAPI) pin narrow Starlette ranges for this reason; if you depend on Starlette directly, pin it and read release notes before bumping rather than floating the version.

## When to Use / When Not

**Use when:**
- You want an async Python web service and full control over the stack above the transport.
- You are building a framework, gateway, or thin service where FastAPI's validation/OpenAPI layer would be dead weight.
- You need WebSockets, streaming responses, or ASGI middleware composition as first-class primitives.
- You want a small, auditable dependency surface.

**Avoid when:**
- You want request/response validation, serialization, and API docs out of the box — use FastAPI, which gives you those on the same foundation.
- You want batteries included (ORM, admin, auth, migrations) — Django covers that ground.
- Your app is synchronous and has no async need — Flask/Django are simpler and the async tax buys nothing.
- Your team is not comfortable reasoning about event-loop blocking and middleware ordering.

## Alternatives

- tiangolo/fastapi — built directly on Starlette; use it (not raw Starlette) the moment you want validation, serialization, and OpenAPI.
- litestar-org/litestar — batteries-included ASGI framework with DI, validation, and plugins baked in; use when you want more than a toolkit but a different opinion set than FastAPI.
- django/django — use when you want a full sync-first framework with ORM, admin, and auth included.
- pallets/flask — use for simple synchronous WSGI services without async requirements.
- sanic-org/sanic — use when you want an all-in-one async framework that ships its own server.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018 | First release by Tom Christie under the `encode` org; ASGI toolkit.[^1] |
| — | 2021 | `TestClient` migrated from `requests` to `httpx`; anyio adopted for asyncio/trio support.[^3] |
| — | (see releases) | Ongoing 0.x line; downstream frameworks pin narrow ranges due to occasional breaking minors. |
| — | — | Maintainership moved to Marcelo Trylesinski (`Kludex`); repo now `Kludex/starlette`.[^1] |

## References

[^1]: Starlette source repository (redirected from `encode/starlette`). https://github.com/Kludex/starlette
[^2]: FastAPI documentation — built on Starlette and Pydantic. https://fastapi.tiangolo.com/
[^3]: AnyIO — async abstraction over asyncio and trio, used by Starlette. https://anyio.readthedocs.io/
[^4]: Starlette middleware documentation (pure ASGI vs `BaseHTTPMiddleware`). https://starlette.dev/middleware/

## Tags

python, asgi, async, web-framework, http, websockets, middleware, toolkit, fastapi-foundation, uvicorn
