# getsentry/responses

> A test utility that mocks the Python `requests` library by patching its transport adapter — no network, no live server.

[GitHub repo](https://github.com/getsentry/responses) ·
[PyPI](https://pypi.org/project/responses/) ·
[License: Apache-2.0](https://github.com/getsentry/responses/blob/master/LICENSE)

## Overview

`responses` intercepts calls made through the `requests` library and returns pre-registered fake responses instead of hitting the network. It was created by David Cramer in 2013[^1] and is maintained under the Sentry (getsentry) organization. The API deliberately mirrors `requests`: you register a URL/method/body triple, run the code under test, and any matching outbound call is served from the registry rather than a socket.

The single most important fact about `responses` is the boundary of what it mocks: it patches `requests.adapters.HTTPAdapter.send`, so it only intercepts traffic that goes through the `requests` library[^2]. Code that reaches for `urllib3`, `http.client`, `httpx`, or `aiohttp` directly passes straight through untouched — a `ConnectionError` or a real network call, not a mock. This tight coupling to `requests` is both why the library is small and predictable and why it is the wrong tool the moment your stack moves off `requests`.

It is aimed at unit and integration tests where you want deterministic HTTP behavior without standing up a fake server. As of 2026 it requires Python 3.8+ and `requests >= 2.30.0`[^3], and remains one of the most widely used HTTP-mocking libraries in the Python test ecosystem.

## Getting Started

```bash
pip install responses
```

```python
import responses
import requests


@responses.activate
def test_fetch():
    responses.get(
        "http://example.com/api/user",
        json={"id": 1, "name": "ada"},
        status=200,
    )

    resp = requests.get("http://example.com/api/user")

    assert resp.json()["name"] == "ada"
    assert responses.calls[0].request.url == "http://example.com/api/user"
```

Any request that does not match a registered response raises `requests.exceptions.ConnectionError`, which makes accidental live traffic in tests fail loudly instead of leaking out.

## Architecture / How It Works

`responses` does not run a server or bind a socket. When `@responses.activate` (or the `RequestsMock` context manager) is entered, it monkeypatches `HTTPAdapter.send` — the point where `requests` hands a `PreparedRequest` to its transport layer[^2]. The patched `send` walks a registry of registered `Response` objects, runs each candidate through its matchers, and returns the first match as a synthetic `requests.Response`. On exit the original adapter method is restored.

Key moving parts:

- **Registry** — by default a `FirstMatchRegistry`. It returns the first matching response; if several responses match the same request, the matched one is consumed and removed so the next call falls through to the following match. `OrderedRegistry` instead ties responses strictly to insertion/invocation order, and you can subclass `registries.FirstMatchRegistry` to implement custom lookup[^4].
- **Matchers** — the `responses.matchers` module supplies composable predicates (`json_params_matcher`, `urlencoded_params_matcher`, `query_param_matcher`, `header_matcher`, `multipart_matcher`, `request_kwargs_matcher`, `fragment_identifier_matcher`). Each is a callable returning `(matched: bool, reason: str)` and receives a `PreparedRequest` augmented with extra `params` and `req_kwargs` attributes.
- **Response body** — can be a string, bytes, a JSON object (auto-sets `Content-Type`), a callback (`CallbackResponse`), or an `Exception` instance, which is raised to simulate transport-level failures.
- **Passthrough** — `add_passthru` / `passthru_prefixes` let selected URL prefixes reach the real network while everything else stays mocked.

Because interception happens at the adapter layer, the full `requests` machinery above it — sessions, auth, retries config, `PreparedRequest` construction, cookie jars — runs for real. That is why matchers see the actually-serialized body and headers, and it is also why anything below or beside the adapter (a custom transport, or a different HTTP library) is invisible to `responses`.

## Production Notes

This is test-only tooling, but the operational footguns are real and recurring:

- **Wrong library = silent no-op.** If the code under test uses `httpx`, `aiohttp`, `urllib3`, or `urllib` directly, `responses` mocks nothing. This is the number-one surprise. Use `respx` for `httpx` and `vcrpy`/`HTTPretty` for cross-library coverage.
- **`assert_all_requests_are_fired` defaults to True.** With `RequestsMock` (and the pytest fixture), any registered response that is never hit raises at teardown. Tests that over-register mocks fail in confusing ways; pass `assert_all_requests_are_fired=False` when you intend some responses to be optional.
- **First-match consumption is order-sensitive.** With the default registry, registering multiple responses for one URL means each call pops one. Tests that assume a fixed response for repeated calls to the same URL either need a single non-consumed response, a callback, or `OrderedRegistry` — and reasoning about "reverse order" behavior trips people up.
- **Registering full URLs with query strings is deprecated.** The old `match_querystring` behavior is deprecated; put query matching in `matchers.query_param_matcher` / `query_string_matcher` and keep the bare path in `url`[^5].
- **Module-level state moved.** `responses.assert_all_requests_are_fired`, `responses.passthru_prefixes`, and `responses.target` were deprecated in 0.20.0 in favor of `responses.mock.*`; several matchers moved out of the top-level namespace into `responses.matchers` in 0.14.0[^5]. Old tutorials cite the removed paths.
- **Not designed for concurrency.** The global `responses.mock` object and its patch of a shared adapter method make parallel use within a process fragile; keep it inside single-threaded test bodies.
- **Strict header matching needs a PreparedRequest.** Because `requests` injects its own default headers, `header_matcher(..., strict_match=True)` will reject ordinary calls; you have to build and send a `PreparedRequest` with overwritten headers to satisfy it.

## When to Use / When Not

**Use when:**
- Your code makes HTTP calls through `requests` and you want fast, deterministic, network-free tests.
- You want to assert on outbound request bodies, params, or headers, not just stub responses.
- You want unmatched requests to fail hard rather than silently hit production.

**Avoid when:**
- The code uses `httpx`, `aiohttp`, or raw `urllib3`/`http.client` — `responses` won't see those calls.
- You want to record and replay real API interactions as fixtures — reach for `vcrpy`.
- You need library-agnostic mocking at the socket layer — `HTTPretty` or a real fake server fits better.
- You're testing async HTTP — `responses` is synchronous, `requests`-only.

## Alternatives

- jamielennox/requests-mock — the other established `requests` mocker; adapter-based like `responses` but with a transport-adapter/fixture-centric API. Use it when you prefer its `Adapter`/matcher style.
- lundberg/respx — mocking for `httpx` (sync and async). Use it when your client is `httpx`, not `requests`.
- kevin1024/vcrpy — records real HTTP interactions to YAML "cassettes" and replays them. Use it when you want fixtures captured from a live API rather than hand-written stubs.
- gabrielfalcao/HTTPretty — socket-level interception that works across HTTP libraries. Use it when you must mock a client `responses` can't reach, and can accept a more invasive approach.
- pytest-dev/pytest-httpserver — a real local WSGI server for tests. Use it when you want to exercise the actual network path against a controllable endpoint.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2013-11-26 | Initial release[^1]. |
| 0.10.0 | 2018-10-18 | Matchers, callbacks, and API maturation era. |
| 0.13.0 | 2021-03-17 | `responses.matchers` and registry groundwork. |
| 0.14.0 | 2021-09-10 | Param matchers moved under `responses.matchers`[^5]. |
| 0.17.0 | 2022-01-10 | `match_querystring` deprecated for explicit query matchers[^5]. |
| 0.20.0 | 2022-03-18 | Top-level `assert_all_requests_are_fired`/`passthru_prefixes`/`target` moved to `responses.mock`[^5]. |
| 0.25.0 | 2024-02-13 | Ongoing matcher/registry and typing refinements. |
| 0.26.0 | 2026-02-19 | Recent maintenance line; requires `requests >= 2.30.0`[^3]. |
| 0.26.2 | 2026-07-03 | Latest release at time of writing. |

## References

[^1]: Repository history, getsentry/responses — first commit 2013-11-15, `0.1.0` tag 2013-11-26. https://github.com/getsentry/responses/tags
[^2]: Mechanism (patching `HTTPAdapter.send`) described in the library source and README "Basics"/"Custom Registry" sections. https://github.com/getsentry/responses/blob/master/README.rst
[^3]: README requirements note — "Responses requires Python 3.8 or newer, and requests >= 2.30.0". https://github.com/getsentry/responses/blob/master/README.rst
[^4]: README, "Response Registry" (FirstMatchRegistry / OrderedRegistry / custom registry). https://github.com/getsentry/responses/blob/master/README.rst
[^5]: README, "Deprecations and Migration Path" table. https://github.com/getsentry/responses/blob/master/README.rst

## Tags

python, testing, http-mocking, requests, unit-testing, test-fixtures, mocking, pytest, api-testing, developer-tools
