# falconry/falcon

> A minimalist, no-magic WSGI/ASGI framework for building REST APIs and microservices in Python.

[GitHub repo](https://github.com/falconry/falcon) ·
[Official website](https://falconframework.org) ·
[Docs](https://falcon.readthedocs.io) ·
[License: Apache-2.0](https://github.com/falconry/falcon/blob/master/LICENSE)

## Overview

Falcon is a low-level web framework for Python whose entire design premise is
subtraction: it does as little as possible per request and pushes almost every
other decision — ORM, serialization schema, validation, auth, templating,
server — onto the developer. First released in 2013 by Kurt Griffiths while at
Rackspace[^1], it targets the operators of large-scale HTTP APIs who care about
predictable latency, a small attack surface, and interfaces that do not break
between releases.

The defining trait is that Falcon has **no dependencies outside the standard
library**[^2]. There is no dependency tree to audit, no transitive breaking
change, and no framework "magic" (no import-time global app, no thread-local
request, no decorator-based routing). Requests map to long-lived resource-class
instances whose methods are named for HTTP verbs (`on_get`, `on_post`). This
makes control flow easy to trace but means Falcon ships none of the ergonomics
that make FastAPI or Django productive out of the box.

The central tension is minimalism versus batteries. Falcon is deliberately not
competing with FastAPI on developer velocity or auto-generated OpenAPI; it
competes on being the thin, stable, fast layer you reach for when a service is
mission-critical and you want to own every abstraction above the socket. That
is a narrower audience than the framework's age and maturity might suggest.

## Getting Started

```bash
pip install falcon
pip install uvicorn      # any ASGI server; or gunicorn/uwsgi for WSGI
```

```python
# app.py — ASGI (falcon.asgi.App); drop `async` and use falcon.App for WSGI
import falcon
import falcon.asgi


class QuoteResource:
    async def on_get(self, req, resp):
        resp.media = {
            "quote": "I've always been more interested in the future.",
            "author": "Sylvia Earle",
        }


app = falcon.asgi.App()
app.add_route("/quote", QuoteResource())
```

```bash
uvicorn app:app        # then: curl localhost:8000/quote
```

Routing uses URI templates with typed field converters, e.g.
`app.add_route("/users/{user_id:int}", UserResource())`; the converted value is
passed as a keyword argument to the responder.

## Architecture / How It Works

**Two separate apps, not one.** `falcon.App` is WSGI (sync); `falcon.asgi.App`
is ASGI (async, added in 3.0)[^3]. They are distinct classes and you cannot mix
sync `on_get` and async `on_get` in the same app — responders must match the
app's execution model. Porting a WSGI service to ASGI is a real migration, not a
flag.

**Compiled router.** The default router does not walk a list of routes at
request time. At `add_route()` it generates Python source for a decision tree
over the URI template set and `exec`s it into a function[^4]. Lookups are close
to a hand-written branch tree, which is a large part of Falcon's throughput
advantage — at the cost that routes are meant to be registered at startup, not
churned dynamically per request.

**Resources and responders.** A resource is an ordinary class instance,
constructed once and reused for every request. There is no per-request object
allocation for the handler and no dependency-injection container. State you want
per request lives on `req.context` / `resp.context`.

**Middleware and hooks.** Cross-cutting logic is either a middleware component
(`process_request`, `process_resource`, `process_response`, plus
`process_startup`/`process_shutdown` and WebSocket variants on ASGI) or a
`@falcon.before` / `@falcon.after` hook on a responder. Middleware runs in a
strict, documented order; response middleware runs in reverse.

**Media handling.** `req.get_media()` / `resp.media` parse and serialize bodies
through pluggable handlers keyed by Internet media type. JSON ships by default;
multipart, URL-encoded form, and MessagePack handlers are included, and the JSON
handler is swappable (e.g. for `orjson`) to change performance characteristics.
Body parsing is opt-in and never happens implicitly.

**Errors.** You raise `falcon.HTTPError` subclasses (`HTTPBadRequest`,
`HTTPUnauthorized`, …) or register handlers with `add_error_handler`. Unhandled
exceptions are not swallowed — a stated design goal is that inputs map
transparently to outputs.

**Cython.** Under any PEP 517 installer Falcon compiles itself with Cython when a
compiler is available, and publishes pre-built wheels; otherwise it falls back to
a pure-Python wheel. The C extension is a speed optimization, not a requirement.

## Production Notes

**It is genuinely "bring your own everything."** No ORM, no request-body schema
validation, no auth, no OpenAPI generation, no settings system. For a small
service this is liberating; for a large team it means you are standardizing
those layers yourself, and two Falcon services in one org can look nothing alike.

**Sync vs async is a hard boundary.** Because WSGI and ASGI are separate app
classes, an async responder that accidentally calls a blocking library will stall
the event loop with no framework guardrail. Choose the execution model up front;
retrofitting is a rewrite of every responder and middleware.

**Stability is a real feature.** The project commits to SemVer with a 100%
test-coverage gate[^2], and breaking changes are confined to major versions that
arrive years apart (2.0 → 3.0 → 4.0 span roughly 2019–2024). Upgrades are
low-drama compared to fast-moving frameworks, which is much of why it persists in
long-lived infra.

**Performance is real but bounded by your choices.** Falcon's per-request
overhead is among the lowest of Python frameworks, and PyPy or a faster JSON
handler widens the gap further. But the framework only owns routing and
request/response plumbing; your serialization, DB driver, and I/O model dominate
real-world latency, and Falcon does nothing to optimize those for you.

**Testing is first-class.** `falcon.testing` provides a client with
`simulate_get`/`simulate_post` and helpers that exercise the app in-process
without a live server — one of the smoother parts of the developer experience.

**Version floor moves rarely but does move.** Recent 4.x requires modern CPython
(3.9+ per current README) and PyPy 3.9+; pinning an old Python can strand you on
an older Falcon line.

## When to Use / When Not

**Use when:**
- You are building REST APIs / microservices where latency and a minimal
  dependency footprint matter more than scaffolding speed.
- You want to own serialization, validation, and auth explicitly rather than
  inherit framework conventions.
- You value API stability and infrequent breaking changes over a fast feature
  cadence.
- You are deploying to PyPy, or otherwise squeezing per-request overhead.

**Avoid when:**
- You want automatic request validation and OpenAPI docs out of the box — that
  is FastAPI's job, not Falcon's.
- You need a full-stack framework with ORM, admin, and templating (Django).
- The team prizes rapid prototyping and a large plugin ecosystem over control.
- Your app is a monolith with server-rendered HTML rather than an API surface.

## Alternatives

- tiangolo/fastapi — pydantic-based validation and auto OpenAPI; use instead when developer velocity and typed request models outweigh minimalism.
- encode/starlette — async-first ASGI toolkit (the layer under FastAPI); use when you want minimalism but live in the uvicorn/Starlette ecosystem.
- pallets/flask — sync microframework with a huge extension ecosystem; use when you want small-but-batteries-available and community plugins.
- django/django — full-stack with ORM/admin/templates; use when you need everything included, not a bare API layer.
- bottlepy/bottle — single-file zero-dependency micro-framework; use for tiny scripts or embedded servers where even Falcon is more than you need.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-01 | Initial release (Kurt Griffiths / Rackspace)[^1]. |
| 1.0 | 2016-04 | First stable release; router and hooks solidified. |
| 2.0 | 2019-02 | Dropped Python 2; pluggable media handlers[^5]. |
| 3.0 | 2021-03 | ASGI, native asyncio, and WebSocket support[^3]. |
| 4.0 | 2024-11 | Modern-Python floor, cleanup of legacy APIs[^6]. |

## References

[^1]: Falcon — official site and project history. https://falconframework.org
[^2]: Falcon README, "Reliable" — no external dependencies, 100% test coverage, SemVer. https://github.com/falconry/falcon#how-is-falcon-different
[^3]: Falcon 3.0 changelog — ASGI / asyncio / WebSocket support. https://falcon.readthedocs.io/en/stable/changes/3.0.0.html
[^4]: Falcon docs — routing and the default compiled router. https://falcon.readthedocs.io/en/stable/api/routing.html
[^5]: Falcon 2.0 changelog. https://falcon.readthedocs.io/en/stable/changes/2.0.0.html
[^6]: Falcon 4.0 changelog. https://falcon.readthedocs.io/en/stable/changes/4.0.0.html

## Tags

python, web-framework, rest-api, wsgi, asgi, microservices, http, async, minimalist, api
