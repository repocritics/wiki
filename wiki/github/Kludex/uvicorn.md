# Kludex/uvicorn

> An ASGI web server for Python — the default runtime under FastAPI and Starlette.

[GitHub repo](https://github.com/Kludex/uvicorn) ·
[Official website](https://uvicorn.dev) ·
[License: BSD-3-Clause](https://github.com/Kludex/uvicorn/blob/main/LICENSE.md)

## Overview

Uvicorn is a minimal ASGI server for Python. ASGI is the async successor to WSGI: instead of a single synchronous callable, an application is an `async` callable over a connection scope with `receive`/`send` channels, which lets one server handle long-lived connections (WebSockets, long-poll, streaming) that WSGI could not[^1]. Uvicorn implements the server half of that contract and speaks HTTP/1.1 and WebSockets. It does **not** implement HTTP/2.

The project was created by Tom Christie as part of the `encode` family (Starlette, HTTPX, Django REST Framework) around 2017, and is the de facto runtime that FastAPI and Starlette tell you to run. Maintenance and the canonical repository have since moved to Marcelo Trylesinski (`Kludex`) — the old `encode/uvicorn` URL now redirects to `Kludex/uvicorn`[^2]. As of 2026 it is one of the most-depended-on servers in the Python async ecosystem, pulled in transitively by nearly every FastAPI deployment.

Its defining tradeoff is deliberate smallness. Uvicorn is a *server*, not a *process manager*: a bare `uvicorn app:app` is a single process with no supervision, no HTTP/2, and no built-in restart-on-crash. That minimalism is why it is fast and easy to embed, and also why production deployments almost always wrap it in something else (Gunicorn, a container orchestrator, or its own `--workers` flag).

## Getting Started

```shell
pip install uvicorn            # pure-Python asyncio + h11
pip install 'uvicorn[standard]'  # adds uvloop + httptools + websockets + watchfiles
```

```python
# example.py — a raw ASGI app (usually you'd use FastAPI/Starlette instead)
async def app(scope, receive, send):
    assert scope["type"] == "http"
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({"type": "http.response.body", "body": b"Hello, world!"})
```

```shell
uvicorn example:app --host 0.0.0.0 --port 8000
uvicorn example:app --reload          # dev only; uses watchfiles
uvicorn example:app --workers 4       # 4 independent worker processes
```

## Architecture / How It Works

Uvicorn is a thin, swappable layer over asyncio. Two components are pluggable via extras, and the `[standard]` install swaps the pure-Python defaults for compiled ones:

- **Event loop** — `asyncio` by default; `uvloop` (a Cython libuv binding) when installed and the platform supports it. uvloop is not available on Windows, where it silently falls back to asyncio.
- **HTTP protocol** — `h11` (pure Python) by default; `httptools` (a Cython wrapper over the Node.js `llhttp` parser) with `[standard]`. Selectable with `--http`.
- **WebSocket protocol** — the `websockets` library by default under `[standard]`; `wsproto` is an alternative you must install and select manually with `--ws`.

The core loop accepts a connection, parses the request with the chosen HTTP implementation, builds the ASGI `scope`, and drives the application coroutine with `receive`/`send` callables backed by asyncio transports. Uvicorn also implements the ASGI **lifespan** protocol (startup/shutdown events) and the WebSocket sub-protocol of ASGI.

Multi-process mode (`--workers N`) uses `multiprocessing` to fork/spawn N copies that share a listening socket via `SO_REUSEPORT`-style socket passing. Workers share **nothing** — no memory, no in-process cache, no coordination. There is a supervising parent that restarts a worker if it dies, but there is no rolling reload of application code and no shared state layer.

Historically Uvicorn shipped a Gunicorn worker class at `uvicorn.workers.UvicornWorker`, letting Gunicorn provide process management while Uvicorn provided the ASGI loop. That integration has been extracted into a separate `uvicorn-worker` package and the in-tree worker is deprecated[^3]; new Gunicorn-based deployments should depend on `uvicorn-worker`.

## Production Notes

- **It is not a process manager.** A single `uvicorn` process that segfaults or OOMs stays dead. In production you want one of: `--workers N` (Uvicorn's own supervisor), Gunicorn + `uvicorn-worker`, or a container platform (Kubernetes, systemd, ECS) that restarts the process. Gunicorn is still commonly preferred when you want mature signal handling, graceful timeouts, and per-worker recycling.
- **No HTTP/2.** Uvicorn serves HTTP/1.1 only. If you need HTTP/2 end-to-end, terminate it at a reverse proxy (nginx, Envoy, Caddy) and let it speak HTTP/1.1 to Uvicorn, or use Hypercorn/Granian instead. HTTP/2 is a recurring "why doesn't this work" issue.
- **Behind a proxy, set proxy headers.** Without `--proxy-headers` (on by default in recent versions) and a correct `--forwarded-allow-ips`, `scope["client"]` and the request scheme reflect the proxy, not the real client. Misconfiguring `--forwarded-allow-ips` to trust everything is a spoofing risk; set it to your proxy's address.
- **`--reload` is development-only.** It watches the filesystem (via `watchfiles`) and restarts on change. It adds overhead and is not safe for production. Never combine `--reload` with `--workers`.
- **uvloop vs asyncio is a real perf gap** for connection-heavy workloads, but it is a native dependency; on Windows or minimal images you run on plain asyncio. Benchmark on your target platform rather than assuming `[standard]` is present.
- **Workers don't share state.** Any in-memory cache, rate limiter, or WebSocket pub/sub must move to an external store (Redis, a message broker) once `--workers > 1`. This surprises people migrating from a single-process dev setup.
- **Graceful shutdown** honors `SIGTERM`/`SIGINT` and drains in-flight requests, but long-lived WebSocket connections may need an explicit `--timeout-graceful-shutdown` or they block the shutdown.

## When to Use / When Not

**Use when:**
- You are running FastAPI or Starlette — this is the intended, best-supported server.
- You want a small, fast ASGI/HTTP-1.1/WebSocket server to sit behind a reverse proxy or orchestrator.
- You need to embed a server programmatically (`uvicorn.run(...)` or the `Server`/`Config` API) inside a larger process.

**Avoid when:**
- You need HTTP/2 or HTTP/3 spoken directly by the app server — use Hypercorn or Granian, or terminate the protocol at a proxy.
- You need a `trio`-based stack — Uvicorn is asyncio-only; Hypercorn supports trio.
- You are deploying to AWS Lambda / API Gateway — you don't want a long-running server at all; use an ASGI-to-Lambda adapter like Mangum.
- You want built-in worker recycling, memory limits, and battle-tested process supervision as a single tool — pair with Gunicorn or run under an orchestrator.

## Alternatives

- pgjones/hypercorn — use instead when you need HTTP/2, HTTP/3, or a `trio` event loop.
- django/daphne — use instead when you're on Django Channels; it's the original ASGI server and supports HTTP/2.
- emmett-framework/granian — use instead when you want a Rust-based server with HTTP/2, TLS, and stronger built-in multi-worker performance.
- benoitc/gunicorn — not an alternative but a complement: run it with `uvicorn-worker` when you want mature process management around Uvicorn.
- jordaneremieff/mangum — use instead when the target is AWS Lambda + API Gateway rather than a persistent server.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017 | Created by Tom Christie under the `encode` org; ASGI HTTP/1.1 + WebSockets[^1]. |
| 0.x | 2017– | Project has deliberately stayed pre-1.0 throughout its life; API is stable in practice despite the version number. |
| — | ~2024 | Gunicorn worker extracted to the separate `uvicorn-worker` package; in-tree worker deprecated[^3]. |
| — | ~2025 | Canonical repo moved from `encode/uvicorn` to `Kludex/uvicorn` (redirect in place)[^2]. |

## References

[^1]: Uvicorn documentation and README, "An ASGI web server, for Python" / "Why ASGI?". https://uvicorn.dev and the ASGI spec at https://asgi.readthedocs.io/en/latest/
[^2]: GitHub repository `Kludex/uvicorn`; `encode/uvicorn` resolves here via redirect. https://github.com/Kludex/uvicorn
[^3]: `uvicorn-worker` — Gunicorn worker class extracted from Uvicorn core. https://github.com/Kludex/uvicorn-worker

## Tags

python, asgi, web-server, asyncio, http, websockets, fastapi, starlette, uvloop, async, http-server
