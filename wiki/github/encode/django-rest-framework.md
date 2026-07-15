# encode/django-rest-framework

> The default toolkit for building Web APIs on top of Django — serializers, viewsets, and a browsable API layered over Django's request/response cycle.

[GitHub repo](https://github.com/encode/django-rest-framework) ·
[Official website](https://www.django-rest-framework.org) ·
[License: BSD (custom, SPDX NOASSERTION)](https://github.com/encode/django-rest-framework/blob/main/LICENSE.md)

## Overview

Django REST framework (DRF) is a library that adds HTTP API primitives to Django: content negotiation, serialization/deserialization, authentication, permissions, throttling, and a self-documenting browsable API. It has been the de facto standard for Django APIs for over a decade and is maintained by Tom Christie under the `encode` organization[^1]. The current major line, DRF 3.x, dates to a 2014 rewrite of the serializer system[^2].

DRF's defining design decision is the **serializer**: a single class that handles both validation of incoming data and representation of outgoing data, modeled deliberately after Django's `Form`/`ModelForm`. This makes CRUD APIs over Django models extremely compact — a `ModelSerializer` plus a `ModelViewSet` plus a router registration is a full REST resource in a dozen lines. The same choice is the source of most of DRF's production friction: serializers are where validation, database access, and output shaping all collide, and they are the first thing to become a performance and complexity bottleneck as an API grows.

The framework is broad and stable rather than fast-moving. It targets Python 3.10+ and Django 4.2 through 6.0[^3], follows Django's own support windows, and iterates conservatively — breaking changes are rare and telegraphed. The tradeoff is that DRF carries design assumptions from the pre-async, pre-OpenAPI era, and modern needs (async views, first-class OpenAPI 3, type hints) are served by ecosystem packages or by looking outside DRF entirely.

## Getting Started

```bash
pip install djangorestframework
```

Add it to `INSTALLED_APPS` and define a resource with a serializer, a viewset, and a router:

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "rest_framework",
]

# urls.py
from django.contrib.auth.models import User
from django.urls import include, path
from rest_framework import routers, serializers, viewsets


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "username", "email", "is_staff"]


class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer


router = routers.DefaultRouter()
router.register(r"users", UserViewSet)

urlpatterns = [
    path("", include(router.urls)),
    path("api-auth/", include("rest_framework.urls")),  # browsable API login
]
```

This yields a full list/create/retrieve/update/delete endpoint set plus an interactive HTML browsable API at the same URLs.

## Architecture / How It Works

DRF sits on top of Django's WSGI/ASGI request cycle and wraps its core objects:

- **`Request`** — wraps Django's `HttpRequest` and adds unified `.data` (parsed body regardless of content type) and `.query_params`. Parsing is deferred and driven by the request's `Content-Type` via configured **parsers** (JSON, form, multipart).
- **`Response`** — holds unrendered data; the actual serialization to bytes happens later via a **renderer** chosen by content negotiation (`Accept` header). The `BrowsableAPIRenderer` is what produces the HTML explorer.
- **`APIView`** — the base class replacing Django's `View`. It wires exception handling (DRF exceptions map to HTTP status codes), authentication, permission checks, and throttling into `dispatch()` before your handler runs.

Above `APIView` sit **generic views** and **viewsets**. `GenericAPIView` + mixins (`ListModelMixin`, `CreateModelMixin`, …) supply the CRUD verbs; `ModelViewSet` bundles all of them and is bound to URLs by a **router** that generates the URL conf from the viewset's queryset and actions.

The **serializer** is the center of gravity. `Serializer.is_valid()` runs field-level and object-level validation and populates `.validated_data`; `.data` produces the outbound representation. `ModelSerializer` introspects a model to generate fields automatically. Serializers are where relations are resolved, meaning a naive list serializer will issue one query per related object per row — the classic **N+1 query** problem, addressed by tuning the viewset's `queryset` with `select_related`/`prefetch_related`, not by changing the serializer.

Cross-cutting concerns are pluggable class lists, configured globally in the `REST_FRAMEWORK` settings dict or per-view: `authentication_classes` (Session, Token, Basic; JWT via third-party), `permission_classes`, `throttle_classes`, `pagination_class`, `filter_backends`. This composition-by-class-list is DRF's core extension mechanism and is consistent across the framework.

## Production Notes

**Serializers are the performance ceiling.** For large or deeply nested payloads, serializer instantiation and field-by-field processing dominate response time. Mitigations: prune `fields`, avoid `SerializerMethodField` that triggers queries, always pair nested serializers with `prefetch_related`, and for read-heavy hot paths consider bypassing the serializer entirely with `.values()` + a plain `Response`. Third-party `drf-serializer` alternatives and `orjson` renderers help at the margins.

**No real async support.** DRF views are synchronous. Even under ASGI, DRF's `APIView.dispatch` is sync and runs in a threadpool; you cannot write `async def` handlers or use async ORM calls inside a DRF view without workarounds[^4]. Projects that need genuine async throughput often reach for Django Ninja or a separate ASGI framework rather than fighting DRF. This is the single most consequential architectural limitation as of 2026.

**OpenAPI generation is weak out of the box.** DRF's built-in schema generation produces incomplete OpenAPI and is effectively in maintenance mode. The community standard is the third-party `drf-spectacular`, which most teams add on day one for usable schemas and Swagger UI.

**Writable nested serializers are a known pain point.** DRF deliberately does not auto-implement `create()`/`update()` for nested writable relations — you must override them. Complex nested writes are where teams write the most bespoke serializer code and hit the most bugs.

**Throttling is naive.** The built-in throttles are cache-backed counters keyed by user or IP; they are approximate, not a substitute for a real rate limiter or API gateway at scale, and their accuracy depends entirely on the shared cache backend.

**Upgrades are usually calm, with one historical exception.** The 2.x → 3.0 serializer rewrite (2014) was a hard break in serializer semantics and required rewriting serializer code[^2]. Since then 3.x upgrades have been low-drama, mostly tracking Django/Python version support. Pin `djangorestframework` and read the release notes, but expect far less churn than most web frameworks.

## When to Use / When Not

**Use when:**
- You already run Django and want ORM-backed REST resources with minimal code.
- You value the browsable API for internal tooling and developer onboarding.
- You need pluggable auth/permissions/throttling that integrate with `django.contrib.auth`.
- Your workload is standard synchronous request/response CRUD.

**Avoid when:**
- You need async views, streaming, or async ORM access as a primary requirement.
- You want first-class OpenAPI 3 and type-hint-driven schemas without add-ons (Django Ninja or FastAPI fit better).
- Your API is a thin pass-through where DRF's serializer overhead buys you little.
- You are not on Django at all — DRF is inseparable from it.

## Alternatives

- vitalik/django-ninja — async-first, type-hint and Pydantic based, auto OpenAPI; use instead when you want modern async Django APIs without serializer classes.
- fastapi/fastapi — full non-Django async framework with Pydantic and OpenAPI built in; use when you are not committed to the Django ORM/admin.
- encode/starlette — the ASGI toolkit under FastAPI; use when you want to assemble an async API layer yourself.
- wsvincent/drfx or hand-rolled Django `JsonResponse` views — use when the API is tiny and DRF's machinery is overkill.
- strawberry-graphql/strawberry — use instead when a GraphQL schema, not REST, is the right interface for your clients.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2011-01 | Initial release (created by Tom Christie)[^1]. |
| 2.0 | 2012-10 | Major rewrite; established the modern DRF shape[^1]. |
| 3.0 | 2014-12 | Serializer system rewritten; hard breaking change[^2]. |
| 3.9 | 2018-10 | Composable permissions, browsable API tweaks. |
| 3.12 | 2020-10 | Django 3.1 support, schema improvements. |
| 3.14 | 2022-09 | Django 4.1 support. |
| 3.15 | 2024-03 | Django 5.0 support, `UniqueConstraint` handling. |
| 3.16 | 2025 | Python 3.13 / Django 5.1+ support, Python 3.10+ baseline[^3]. |

## References

[^1]: Django REST framework repository and history, `encode/django-rest-framework`. https://github.com/encode/django-rest-framework
[^2]: DRF 3.0 announcement — serializer rewrite. https://www.django-rest-framework.org/community/3.0-announcement/
[^3]: DRF README requirements: Python 3.10+, Django 4.2–6.0. https://www.django-rest-framework.org/
[^4]: DRF documentation on async support and views. https://www.django-rest-framework.org/api-guide/views/

## Tags

python, django, rest-api, web-framework, serialization, api-toolkit, backend, orm, http, openapi
