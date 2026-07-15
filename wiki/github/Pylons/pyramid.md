# Pylons/pyramid

> A "pay only for what you use" Python web framework — the middle ground between a microframework and a batteries-included stack.

[GitHub repo](https://github.com/Pylons/pyramid) ·
[Official website](https://trypyramid.com/) ·
[License: BSD-derived (Repoze Public License)](http://repoze.org/license.html)

## Overview

Pyramid is a WSGI web framework maintained by the Pylons Project. It descends from two older projects that merged in 2010–2011: `repoze.bfg` (Chris McDonough / Agendaless Consulting, Zope-derived) and the original Pylons framework, whose community adopted the merged result under the Pyramid name for its 1.0 release[^1]. GitHub still lists the creation date as 2010; the codebase carries genes from both lineages.

Its defining stance is minimalism-with-optionality: unlike Django it bundles no ORM, no admin, no form library, and no template engine by default, but unlike Flask it ships first-class machinery for URL generation, authorization, renderers, request/response objects, and configuration conflict detection out of the box. The project markets this as "start small, finish big" — the same framework meant to serve a single-file hello-world and a large multi-team application. The cost of that flexibility is a larger surface area of concepts (traversal, the component registry, tweens, renderers) than a microframework, and more assembly than a full-stack framework.

Pyramid is synchronous and WSGI-native. It predates Python's `async`/`await` era and has not pivoted to ASGI; teams needing native async concurrency generally look elsewhere. It remains a stable, conservative choice with an unusually strong record on backward compatibility and documentation completeness.

## Getting Started

```bash
pip install pyramid waitress
```

A minimal single-file application (URL dispatch):

```python
from wsgiref.simple_server import make_server
from pyramid.config import Configurator
from pyramid.response import Response

def hello(request):
    return Response(f"Hello, {request.matchdict['name']}")

with Configurator() as config:
    config.add_route('hello', '/hello/{name}')
    config.add_view(hello, route_name='hello')
    app = config.make_wsgi_app()

make_server('0.0.0.0', 6543, app).serve_forever()
```

Larger projects are typically scaffolded with a cookiecutter, run under `waitress` via a PasteDeploy `.ini` file, and use `@view_config` decorators picked up by `config.scan()` instead of imperative `add_view` calls.

## Architecture / How It Works

Pyramid has **two routing systems that can coexist in one app** — this is its most distinctive and most confusing design decision:

1. **URL dispatch** — a Routes-style ordered list of URL patterns (`add_route`) matched top to bottom. Familiar to anyone coming from Flask, Django, or Rails.
2. **Traversal** — a Zope-derived model where the URL path is walked segment by segment against a graph of "resource" objects, and the located resource plus a view name select the view. Suited to hierarchical, content-management-shaped data and per-object authorization.

Underneath both sits the **Zope Component Architecture (ZCA)**: an application-wide registry (`registry`) of components looked up by interface. View lookup, renderers, subscribers, and adapters all resolve through it. This is powerful and rarely seen elsewhere in Python web frameworks, but it is also the source of Pyramid's learning curve — much behavior is registered indirectly rather than wired explicitly.

Other core pieces:

- **Renderers** — a view returns a plain dict and a renderer (`json`, or a template like Chameleon/Mako/Jinja2) turns it into a response. Decouples view logic from serialization.
- **Configuration phase** — `Configurator` collects registrations and runs a **conflict-detection** pass at commit time, raising if two registrations clash. `config.scan()` performs a `venusian`-based decorator scan. `config.include()` composes add-ons.
- **Tweens** — a middleware-like layer inside the framework (between the router and the view) for cross-cutting concerns; the exception view and transaction management ride here.
- **Request lifecycle** — the `request` object is central, extensible via `add_request_method`, and carries the matched route, matchdict, and registry access.
- **Security** — a pluggable **security policy** governs authenticated identity and permission checks against view-declared `permission=` requirements. In 2.0 this replaced the older split authentication-policy / authorization-policy pair with a single `ISecurityPolicy`[^2].

## Production Notes

**Sync-only, WSGI-only.** Pyramid runs one request per worker thread/process. Concurrency scales by adding workers (waitress threads, or gunicorn/uWSGI processes), not by an event loop. For high-fan-out I/O-bound workloads an async framework will use fewer resources. There is no supported ASGI story.

**Server choice.** `waitress` (the Pylons Project's own pure-Python WSGI server) is the documented default and is fine behind a reverse proxy. For higher throughput teams commonly switch to gunicorn or uWSGI; this is a deployment config change, not a code change.

**No ORM, no migrations, no forms bundled.** The SQLAlchemy cookiecutter wires in SQLAlchemy + Alembic + `pyramid_tm` (transaction manager) and `zope.sqlalchemy`, giving a request-scoped transaction that commits on success and rolls back on exception. This transaction-per-request pattern is a genuine strength but is convention, not magic — misunderstanding `pyramid_tm`'s commit/abort triggers (e.g. returning a 4xx does not by itself abort) is a recurring footgun.

**Traversal vs. dispatch confusion.** Teams that pick traversal without needing it pay a comprehension tax; teams that need per-object ACLs and pick pure URL dispatch reimplement authorization awkwardly. Choosing the wrong model early is the most common architectural regret. Default to URL dispatch unless your data is genuinely a tree.

**Backward compatibility.** Pyramid's deprecation discipline is strong — features are deprecated with warnings across multiple minor releases before removal, and the "What's New" / upgrade docs are thorough. The one hard cut was **2.0 dropping Python 2** and reworking the security API; apps using the old `AuthTktAuthenticationPolicy` / ACL-authorization split needed migration to `SecurityPolicy`[^2].

**Ecosystem size.** Add-ons (`pyramid_jinja2`, `pyramid_debugtoolbar`, `pyramid_retry`, `cornice` for REST, etc.) are solid but far fewer and less actively churned than Flask's or Django's. Expect to write more integration glue and find fewer copy-paste answers.

## When to Use / When Not

**Use when:**
- You want more structure than Flask but reject Django's all-or-nothing stack and want to choose your own ORM, templating, and auth.
- Your data is hierarchical and benefits from traversal + per-object authorization (CMS-shaped domains).
- You value long-term stability, backward-compatibility discipline, and complete documentation over ecosystem size or novelty.

**Avoid when:**
- You need native async / high-concurrency I/O — reach for an ASGI framework instead.
- You want batteries included (admin, ORM, migrations, forms) with minimal assembly — Django is a better fit.
- You want the largest possible tutorial/StackOverflow/plugin ecosystem — Flask and Django dominate mindshare.
- The team is small and wants the fastest possible time-to-first-endpoint with the fewest concepts — a microframework is lighter.

## Alternatives

- pallets/flask — simpler microframework, larger community; choose it when you want minimal concepts and the biggest ecosystem, and don't need traversal or built-in conflict detection.
- django/django — batteries-included; choose it when you want ORM, admin, migrations, and auth out of the box rather than assembling them.
- encode/starlette — async/ASGI foundation; choose it when native concurrency for I/O-bound workloads matters.
- tiangolo/fastapi — async, type-hint-driven APIs with automatic OpenAPI; choose it for modern typed REST/JSON services.
- bottle/bottle — single-file micro WSGI framework; choose it for tiny scripts and embedded use where even Flask is too much.

## History

| Version | Date | Notes |
|---------|------|-------|
| repoze.bfg 1.0 | 2009 | Precursor framework by Agendaless Consulting[^1]. |
| Pyramid 1.0 | 2011-01 | repoze.bfg renamed; Pylons community consolidates under Pyramid[^1]. |
| 1.x series | 2011–2018 | Long, backward-compatible minor cadence (URL dispatch + traversal, renderers, tweens). |
| 1.10 | 2018-2019 | Final 1.x line; last releases to support Python 2. |
| 2.0 | 2021-01 | Python 3 only; unified `SecurityPolicy` API replacing authn/authz split[^2]. |

## References

[^1]: Pyramid documentation, "Introduction / project history" — Pylons Project. https://docs.pylonsproject.org/projects/pyramid/en/latest/
[^2]: Pyramid 2.0 upgrading and security API docs — Pylons Project. https://docs.pylonsproject.org/projects/pyramid/en/latest/narr/security.html
[^3]: Pyramid README and Pylons Project overview. https://trypyramid.com/

## Tags

python, web-framework, wsgi, backend, pylons, traversal, url-dispatch, zope-component-architecture, sync, minimalist
