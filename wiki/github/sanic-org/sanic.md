# sanic-org/sanic

> An async Python web framework that ships its own production HTTP server, built for throughput before FastAPI existed.

[GitHub repo](https://github.com/sanic-org/sanic) ·
[Official website](https://sanic.dev) ·
[License: MIT](https://github.com/sanic-org/sanic/blob/main/LICENSE)

## Overview

Sanic is a Python 3.10+ web framework and HTTP server built around `async`/`await`[^1]. It first appeared in 2016, before ASGI was standardized and years before FastAPI, making it one of the earliest async-native Python web frameworks. Unlike Flask (WSGI) or FastAPI (an ASGI application that needs an external server such as Uvicorn), Sanic bundles its own production-oriented HTTP server. You run a Sanic app directly; there is no mandatory gunicorn/uvicorn layer in front of it.

The defining tension is scope. Sanic deliberately stays a server-plus-router: it does not include request validation, serialization, or automatic OpenAPI generation in the core package. Those live in a separate first-party package, `sanic-ext`, and routing itself was extracted into `sanic-routing`[^2]. This keeps the core small and fast but means the out-of-box developer experience is thinner than FastAPI's — you opt into the batteries rather than getting them by default. Sanic also supports an ASGI mode, so it can run under any ASGI server, but its own server is where the performance story lives[^1].

The project is community-maintained under the `sanic-org` organization, using calendar versioning (CalVer) with designated long-term-support releases. It remains actively developed, with commits landing regularly as of mid-2026.

## Getting Started

```bash
pip install sanic
# uvloop + ujson are pulled in for speed; opt out with
# SANIC_NO_UVLOOP=true / SANIC_NO_UJSON=true at install time
```

```python
# server.py
from sanic import Sanic
from sanic.response import json

app = Sanic("my-app")

@app.get("/")
async def handler(request):
    return json({"hello": "world"})
```

```bash
sanic server:app --host 0.0.0.0 --port 8000 --workers 4
```

The `sanic` CLI is the intended entry point; it boots the worker manager, binds the socket, and supervises worker processes. For local iteration, `--dev` enables auto-reload and a debug page.

## Architecture / How It Works

**Own HTTP server.** Sanic parses HTTP itself and runs handlers on an asyncio event loop, defaulting to `uvloop` where available (uvloop has no Windows build, so Windows falls back to the stock loop). This is the historical reason Sanic benchmarked well — there is no WSGI shim and no separate server process in the default path.

**Router.** Routing is handled by `sanic-routing`, a standalone package that compiles the route table into an optimized matcher rather than iterating a list of patterns[^2]. Path parameters carry types (`<id:int>`, `<name:str>`, custom converters). The compiled router is fast but has occasionally surprised users when route-registration order or dynamic segments interact — router changes are a recurring source of minor version churn.

**Worker manager.** Since v22.9, Sanic ships a process manager that forks worker processes, supervises them, supports zero-downtime restarts, and exposes an "Inspector" control channel[^3]. Cross-worker state is coordinated through `app.shared_ctx` (backed by process-shared objects you set up before workers fork). This replaced the older, thinner gunicorn-based multiprocessing story.

**Blueprints and middleware.** Applications are composed from `Blueprint` objects (route groups with their own prefixes and middleware). Request/response middleware, signals (an internal pub/sub event system), and listeners (`before_server_start`, `after_server_stop`, etc.) are the main extension points.

**The `-ext` split.** `sanic-ext` layers on Pydantic/dataclass validation, dependency injection, CORS, and auto-generated OpenAPI docs. It is a separate install and separate release cadence from core Sanic — a deliberate architectural boundary, and a thing to remember when reading tutorials that assume it is present.

## Production Notes

**CalVer and LTS.** Versions are `YY.MM` (e.g. 23.12, 24.12). The December release each year is a long-term-support line; interim releases (`.3`, `.6`, `.9`) move faster and can carry breaking changes. For production, pin to an LTS release unless you specifically need a newer feature — mixing interim releases into a long-lived service invites migration work every quarter.

**ASGI mode differs from native mode.** Running Sanic under an external ASGI server (Uvicorn, Hypercorn, Daphne) is supported, but the worker manager, some lifecycle listeners, and server-level tuning behave differently or not at all — you inherit the ASGI server's process model instead. Decide early which mode you are deploying, because operational docs and behavior diverge between the two.

**uvloop and ujson are optional but assumed.** The default install pulls them in; they are what the performance numbers depend on. If you disable them (or run somewhere they will not build), expect lower throughput and subtly different JSON edge-case behavior (`ujson` is stricter/looser than the stdlib in places).

**Shared state across workers is manual.** There is no implicit shared cache. Multi-worker deployments must use `shared_ctx`, an external store (Redis), or accept per-worker isolation. Newcomers frequently assume module-level globals are shared across workers; they are not.

**Ecosystem size.** This is the honest operator caveat: Sanic's third-party ecosystem (auth, ORM integrations, extensions, Stack Overflow coverage) is materially smaller than Flask's or FastAPI's. Common needs are solvable but often require more first-party wiring. FastAPI overtook Sanic in mindshare around 2019–2020, and much of the async-Python tutorial content targets FastAPI, not Sanic.

## When to Use / When Not

**Use when:**
- You want an async framework that is also its own production server, without an external ASGI server in the path.
- Raw request throughput and a lean core matter more than batteries-included ergonomics.
- You need fine control over the process/worker model, server lifecycle, and low-level HTTP handling.
- You are comfortable adding `sanic-ext` for validation/OpenAPI rather than getting them by default.

**Avoid when:**
- You want automatic request validation and OpenAPI docs out of the box — FastAPI gives that with zero extra packages.
- You need the largest possible ecosystem and community answer coverage — Flask and FastAPI dominate there.
- Your workload is synchronous/CPU-bound or trivially small — async buys little and Flask is simpler.
- You require a batteries-included, ORM-and-admin stack — reach for Django.

## Alternatives

- tiangolo/fastapi — use instead when you want Pydantic validation and auto OpenAPI in the core, and the largest async-Python ecosystem.
- pallets/flask — use instead when the app is synchronous and you value the deepest ecosystem and simplest mental model.
- encode/starlette — use instead when you want a minimal ASGI toolkit to assemble a framework yourself (it is also FastAPI's foundation).
- aio-libs/aiohttp — use instead when you need a combined async HTTP client and server at a lower level of abstraction.
- encode/uvicorn — not a framework but the ASGI server most FastAPI/Starlette apps run on; relevant because Sanic's bundled server is the thing it replaces.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016 | Initial release; early async-native Python framework, pre-ASGI[^1]. |
| 18.12 | 2018-12 | First CalVer release; LTS line begins[^4]. |
| 19.6 | 2019-06 | ASGI support added. |
| 20.12 | 2020-12 | LTS release. |
| 21.3 | 2021-03 | Major internals refactor; `sanic-routing` and `sanic-ext` split out[^2]. |
| 21.12 | 2021-12 | LTS release. |
| 22.9 | 2022-09 | Worker manager + Inspector; new process model[^3]. |
| 22.12 | 2022-12 | LTS release. |
| 23.12 | 2023-12 | LTS release. |
| 24.12 | 2024-12 | LTS release; Python 3.10+ baseline. |

## References

[^1]: Sanic README and documentation. https://github.com/sanic-org/sanic and https://sanic.dev
[^2]: sanic-routing and sanic-ext packages. https://github.com/sanic-org/sanic-routing and https://sanic.dev/en/plugins/sanic-ext/getting-started.html
[^3]: Sanic User Guide, "Worker manager". https://sanic.dev/en/guide/deployment/manager.html
[^4]: Sanic User Guide, "Project policies / versioning". https://sanic.dev/en/org/policies.html

## Tags

python, async, asyncio, web-framework, http-server, asgi, api-server, backend, uvloop, calver, high-throughput
