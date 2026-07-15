# kevin1024/vcrpy

> Record real HTTP interactions once, replay them from a flat file forever — deterministic, offline HTTP tests for Python.

[GitHub repo](https://github.com/kevin1024/vcrpy) ·
[Documentation](https://vcrpy.readthedocs.io/) ·
[License: MIT](https://github.com/kevin1024/vcrpy/blob/master/LICENSE.txt)

## Overview

VCR.py is a Python port of Ruby's VCR library[^1]. The first time a test runs inside a `use_cassette` context (or under the decorator), VCR.py intercepts every HTTP request the code makes, records the request and its response, and serializes both to a flat file called a *cassette* (YAML by default). On every subsequent run it matches outgoing requests against the recorded ones and replays the stored responses, so no real network traffic occurs. The payoff is offline, deterministic, fast tests against third-party HTTP APIs[^2].

The project has existed since 2012 and remains actively maintained (2,976 stars, 428 forks, commits landing through mid-2026). Its ~146 open issues are typical for a library of this age and scope — the long tail is dominated by "cassette won't match my request" support questions and breakage reports tied to new releases of the underlying HTTP clients.

The defining tension is that VCR.py achieves transparency by *monkeypatching the internal connection classes* of the libraries it supports (urllib3, requests, http.client, aiohttp, httpx, tornado, boto3, httplib2). This is what lets you add one decorator and have `requests.get(...)` transparently replay — but it also couples VCR.py to private internals of those libraries. A minor release of urllib3 or httpx can break playback until VCR.py catches up. Convenience at the front, fragility at the seam.

## Getting Started

```bash
pip install vcrpy
```

```python
import vcr
import requests

@vcr.use_cassette("fixtures/vcr_cassettes/synopsis.yaml")
def test_iana():
    response = requests.get("http://www.iana.org/domains/reserved")
    assert "Example domains" in response.text
```

The first run records `synopsis.yaml`; every later run replays it with no network. As a context manager, with explicit config:

```python
my_vcr = vcr.VCR(
    record_mode="once",
    match_on=["method", "scheme", "host", "port", "path", "query"],
    filter_headers=["authorization"],   # keep secrets out of the cassette
)

with my_vcr.use_cassette("fixtures/cassettes/login.yaml"):
    requests.post("https://api.example.com/login", json={"user": "tom"})
```

## Architecture / How It Works

VCR.py does not sit behind a proxy or a public adapter API. At `use_cassette` entry it **patches the connection classes** of each supported client — for example replacing `http.client.HTTPConnection` / `HTTPSConnection` (and the equivalents inside urllib3, aiohttp, httpx) with VCR stub subclasses. Requests routed through those stubs are turned into a normalized `Request` object rather than opening a socket.

For each request the cassette is consulted:

- **Matching.** A request matches a recorded interaction when *all* configured matchers agree. The defaults are `method` and `uri` (which decomposes into scheme, host, port, path, query). Additional matchers — `body`, `headers`, `raw_body`, `query` — are opt-in. Crucially, **request body is not matched by default**, so two POSTs to the same URL with different payloads collide unless you add the `body` matcher[^3].
- **Record mode** decides what happens on a hit or miss[^4]:
  - `once` (default) — record if the cassette file does not exist; otherwise replay and *reject* any request not already recorded.
  - `new_episodes` — replay known requests, record new ones.
  - `none` — replay only; any unmatched request raises. Good for CI.
  - `all` — never replay; always hit the network and re-record.
- **Replay.** On a match, the stored response is deserialized and handed back through the stub as if it came off the wire. `play_count` tracks reuse; by default a recorded interaction can be replayed multiple times unless `allow_playback_repeats` is toggled.
- **Miss under `once`/`none`** raises `CannotOverwriteExistingCassetteException`.

Scrubbing hooks run at record time: `filter_headers`, `filter_query_parameters`, `filter_post_data_parameters`, and the general-purpose `before_record_request` / `before_record_response` callables let you redact or rewrite interactions before they touch disk.

Serialization is pluggable (YAML default, JSON built in, custom serializers registerable). Cassette persistence goes through a `FilesystemPersister` that can be swapped for custom storage.

## Production Notes

**Cassettes leak secrets by default.** A raw recording contains the full request and response, including `Authorization` headers, cookies, session tokens, and any credentials in bodies or query strings. Without `filter_headers` / `before_record_request`, those get committed to your repository. Treat scrubbing as mandatory, not optional, and review the first recording of every new cassette by hand.

**Matching is the number-one support cost.** Because body is unmatched by default, request-heavy suites (especially anything POST-oriented like GraphQL, where every call hits one endpoint) can silently replay the wrong response. Adding the `body` matcher fixes collisions but then breaks on any request carrying a timestamp, nonce, CSRF token, or UUID — you end up writing a custom matcher or a `before_record_request` normalizer. There is no perfect default here; expect to tune `match_on` per project.

**Coupling to HTTP-client internals is the core fragility.** VCR.py patches private connection classes, so major releases of urllib3, requests, aiohttp, or httpx have historically broken playback until VCR.py ships a compatible release. Pin VCR.py and your HTTP client together, and treat a client upgrade as a change that needs its cassettes re-verified. httpx and async (aiohttp) support in particular have been the areas most prone to edge cases — streaming responses, connection pooling, and content decoding.

**Stale cassettes.** VCR.py does not detect that an API changed; a cassette recorded against last year's response replays happily forever, so your "passing" test may assert against a contract that no longer exists. The intended workflow is to periodically delete cassettes and re-record (`record_mode="all"`), which requires live credentials and network — the opposite of the offline promise. Budget for it.

**Repo bloat.** Cassettes are checked-in fixtures. Large JSON/binary responses (especially compressed payloads that VCR.py decodes) inflate diffs and repo size; some teams keep cassettes in a separate directory or LFS.

**Concurrency.** Playback ordering with many similar requests in one cassette is sensitive to `play_count` and repeat settings; heavily parallel code under a single cassette can behave nondeterministically.

## When to Use / When Not

**Use when:**
- You test code that calls third-party HTTP APIs and want offline, deterministic, fast tests.
- You want a near-zero-config way to freeze a real interaction rather than hand-writing mocks.
- Your requests are stable and distinguishable by method + URL.

**Avoid when:**
- You're testing your own server — use the framework's test client (Flask/Django/FastAPI) instead of replaying HTTP.
- Requests are inherently non-deterministic (per-call nonces, signed payloads) and hard to match reliably.
- You need explicit, readable stubbing of a handful of calls — a manual mocker (`responses`, `respx`) is clearer than a recorded blob.
- You cannot risk secrets in fixtures and don't want to maintain scrubbing discipline.

## Alternatives

- getsentry/responses — manual request stubbing for the `requests` library; explicit and readable, no recording, no monkeypatching of internals. Use when you want to hand-write a few responses.
- lundberg/respx — mock/router for `httpx` (sync + async). Use when your stack is httpx-only and you prefer declarative routing over recorded cassettes.
- betamaxpy/betamax — VCR-style record/replay built as a `requests` transport adapter (no internal patching). Use when you want the cassette workflow but only need `requests` and want a cleaner integration seam.
- jamielennox/requests-mock — adapter-based mocking for `requests`. Use for precise, code-defined responses without fixtures.
- kiwicom/pytest-recording — thin pytest layer *on top of* VCR.py. Use it alongside VCR.py, not instead, when you want ergonomic pytest markers/fixtures.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012 | Repo created as a Python port of Ruby's VCR[^1]. |
| 1.x | ~2015 | First stable line; core record/replay + matchers. |
| 4.x | ~2020 | Python 2 support dropped; modernized client patching. |
| 5.x | ~2023 | Continued client-compat maintenance, config refinements. |
| 6.x | ~2024 | Broader httpx/async support and newer-Python baselines. |

## References

[^1]: Ruby's VCR library, the original this project ports. https://github.com/vcr/vcr
[^2]: VCR.py rationale and benefits (offline, deterministic, faster tests). https://vcrpy.readthedocs.io/en/latest/usage.html
[^3]: VCR.py request matching / `match_on` configuration. https://vcrpy.readthedocs.io/en/latest/configuration.html
[^4]: VCR.py record modes (`once`, `new_episodes`, `none`, `all`). https://vcrpy.readthedocs.io/en/latest/usage.html#record-modes

## Tags

python, http, testing, mocking, record-replay, test-fixtures, http-client, pytest, cassettes, integration-testing
