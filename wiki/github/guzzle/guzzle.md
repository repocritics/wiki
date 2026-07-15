# guzzle/guzzle

> The default PHP HTTP client: a PSR-7/PSR-18 request layer with a composable middleware stack over pluggable cURL and stream transports.

[GitHub repo](https://github.com/guzzle/guzzle) ·
[Packagist: guzzlehttp/guzzle](https://packagist.org/packages/guzzlehttp/guzzle) ·
[License: MIT](https://github.com/guzzle/guzzle/blob/7.15/LICENSE)

## Overview

Guzzle is the most widely installed HTTP client in the PHP ecosystem, dating to 2011[^1]. It sends synchronous and asynchronous HTTP requests through a single interface and abstracts away the underlying transport, so application code does not depend directly on cURL, PHP streams, or sockets. Since version 6 it is built on PSR-7 message interfaces, and since version 7 it implements PSR-18 (`ClientInterface::sendRequest`), which lets any PSR-18-aware library treat Guzzle as an interchangeable client[^2].

Guzzle's defining design choice is the **handler-and-middleware stack**: every request passes through an onion of middleware (redirects, retries, cookies, error handling) before reaching a terminal handler that actually performs I/O. This makes behavior highly composable but also means the "client" is a thin object over a mutable stack that most users never inspect. The other defining choice is its **promise-based async model**, which is cooperative rather than event-loop-driven — a distinction that trips up developers expecting Node.js-style concurrency (see Production Notes).

The library is split across three composer packages that version independently: `guzzlehttp/guzzle` (the client), `guzzlehttp/psr7` (message/stream/URI implementations), and `guzzlehttp/promises` (the Promises/A+ implementation). Much of Guzzle's real-world security and maintenance history actually lives in the `psr7` package, not the client itself.

## Getting Started

```bash
composer require guzzlehttp/guzzle
```

```php
<?php
require 'vendor/autoload.php';

use GuzzleHttp\Client;
use GuzzleHttp\Promise\Utils;

$client = new Client([
    'base_uri' => 'https://api.github.com',
    'timeout'  => 5.0,
]);

// Synchronous request.
$res = $client->request('GET', '/repos/guzzle/guzzle');
echo $res->getStatusCode();               // 200
$data = json_decode((string) $res->getBody(), true);

// Concurrent requests — promises only progress inside wait()/unwrap().
$promises = [
    'a' => $client->getAsync('/repos/guzzle/guzzle'),
    'b' => $client->getAsync('/repos/symfony/symfony'),
];
$responses = Utils::unwrap($promises);    // blocks until all settle
```

## Architecture / How It Works

A `Client` holds an options array and a `HandlerStack`. Calling `request()` builds a PSR-7 `Request`, merges per-call options over client defaults, and pushes the request into the stack.

- **Handler** — the terminal function that performs the transport. `CurlHandler` (blocking cURL), `CurlMultiHandler` (async via `curl_multi`), and `StreamHandler` (PHP stream wrappers, no cURL) ship in-box. `Utils::chooseHandler()` picks cURL when the extension is present, otherwise streams. A handler returns a `PromiseInterface`.
- **Middleware** — callables wrapped around the handler in a `HandlerStack`. Defaults pushed by `HandlerStack::create()` include `http_errors` (throw on 4xx/5xx), `redirect` (follow `Location`), `cookies`, and `prepare_body`. You add your own with `push()`, `unshift()`, or named insertion. Each middleware can rewrite the outgoing request and the incoming response.
- **Promises** — `guzzlehttp/promises` implements Promises/A+ without an event loop. There is no reactor thread: a pending promise advances only when something ticks it. For async requests that tick happens inside `wait()` (which drives the `curl_multi` loop) or inside a `Pool`.

Request configuration is entirely option-driven (`RequestOptions`): `json`, `form_params`, `multipart`, `query`, `headers`, `auth`, `proxy`, `verify`, `timeout`, `allow_redirects`, `on_stats`, `sink`, and `stream`. Options are the primary API surface — most behavior is changed by passing an array, not by subclassing.

Concurrency at scale uses `GuzzleHttp\Pool` or `Utils::settle()/unwrap()` to run a bounded set of promises and rendezvous on completion. `EachPromise` powers the `Pool` and caps in-flight requests via a `concurrency` setting.

## Production Notes

**"Async" is not an event loop.** `sendAsync()` returns immediately, but the request does not progress until you call `wait()` (directly or via `Pool`/`unwrap`). A promise you create and never wait on will never resolve, and mixing Guzzle promises with a real event loop (ReactPHP, Amp) does not multiplex them — the blocking `wait()` will stall the loop. If you need genuine non-blocking I/O, use an event-loop-native client instead.

**Error handling is on by default.** With `http_errors` enabled, 4xx responses throw `ClientException` and 5xx throw `ServerException` (both extend `BadResponseException` / `RequestException`). Code that inspects status codes manually must pass `['http_errors' => false]` or wrap calls in try/catch — a frequent surprise when migrating from lower-level clients.

**TLS verification is on by default** (`verify => true`), using the system CA bundle or cURL's. Disabling it (`verify => false`) is a common but dangerous copy-paste fix for cert errors; the correct fix is usually pointing `verify` at a valid CA file.

**Memory and streaming.** Response bodies are streamed, but calling `(string) $res->getBody()` or decoding JSON pulls the whole body into memory. For large downloads use the `sink` option (write to a path/stream) and/or `stream => true` to avoid buffering.

**Security history lives in psr7.** Several notable advisories were in `guzzlehttp/psr7`, not the client — most importantly the 2022 fixes for leaking `Authorization` and `Cookie` headers across hosts on cross-domain redirects, and for improper `Cookie`/multipart parsing[^3]. Keep `guzzlehttp/psr7` updated independently of the client, and audit any middleware that follows redirects to untrusted hosts.

**Upgrade pains.** The 3.x → 4.x jump changed the Composer package and namespace (`guzzle/guzzle`/`Guzzle` → `guzzlehttp/guzzle`/`GuzzleHttp`) and is a rewrite, not a bump. 5.x → 6.x moved to PSR-7 immutable messages (no more mutable request/response setters). 6.x → 7.x added PSR-18 and raised the minimum to PHP 7.2.5. Each of these broke source-level compatibility.

## When to Use / When Not

**Use when:**
- You want the de facto standard client with the largest middleware and integration ecosystem.
- You need PSR-7/PSR-18 interop so the client can be swapped or injected.
- Your workload is request/response with modest, bounded concurrency handled by `Pool`.

**Avoid when:**
- You need true non-blocking, event-loop concurrency (websockets, thousands of streaming connections) — Guzzle's cooperative model will fight your loop.
- You want a dependency-light footprint; Guzzle pulls in `psr7` and `promises` and, in practice, ext-curl.
- You are already on Symfony and want native `HttpClient` streaming/multiplexing without a second abstraction.

## Alternatives

- symfony/http-client — use instead when you want native response streaming and HTTP multiplexing, or already live in Symfony; also PSR-18.
- amphp/http-client — use instead when you need genuine non-blocking concurrency on an Amp/Fiber event loop.
- php-http/httplug — use instead when you want to depend on an abstraction (virtual package) rather than a concrete client.
- kriswallsmith/buzz — use instead when you want a small, straightforward PSR-18 client without a middleware stack.
- WpOrg/Requests — use instead when you want a cURL-optional client with minimal dependencies (the client WordPress core ships).

## History

| Version | Date | Notes |
|---------|------|-------|
| 3.x | ~2013 | Pre-PSR-7 era; package `guzzle/guzzle`, namespace `Guzzle`. EOL 2016-10-31[^1]. |
| 4.x | 2014 | Repackaged as `guzzlehttp/guzzle`, namespace `GuzzleHttp`. Not PSR-7. EOL 2016-10-31. |
| 5.x | 2014 | Refined handler/event model. Not PSR-7. EOL 2019-10-31. |
| 6.0 | 2015 | PSR-7 immutable messages, promise-based async, middleware stack. EOL 2023-10-31. |
| 7.0 | 2020 | PSR-18 client, PHP 7.2.5+ minimum, split promises/psr7 packages[^2]. |
| 7.15 | 2026 | Current 7.x line (default branch), MIT, PHP >=7.2.5,<8.6[^1]. |

## References

[^1]: Guzzle README, "Version Guidance" table (EOL dates, PHP ranges, namespaces). https://github.com/guzzle/guzzle#version-guidance
[^2]: Guzzle documentation and PSR-18 (HTTP Client) specification. https://docs.guzzlephp.org · https://www.php-fig.org/psr/psr-18/
[^3]: GuzzleHttp security advisories (2022 cross-domain header leakage and cookie/multipart parsing fixes, primarily in guzzlehttp/psr7). https://github.com/guzzle/psr7/security/advisories

## Tags

php, http-client, psr-7, psr-18, rest-api, middleware, curl, async, promises, api-integration, networking
