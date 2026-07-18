# haskell/wreq

> A lens-based HTTP client library for Haskell — ergonomic one-liners over
> http-client, in maintenance mode since the late 2010s.

[GitHub repo](https://github.com/haskell/wreq) ·
[Tutorial](http://www.serpentine.com/wreq/) ·
[License: BSD-3-Clause](https://github.com/haskell/wreq/blob/master/LICENSE.md)

## Overview

wreq is an HTTP client library for Haskell by Bryan O'Sullivan (author of
aeson, attoparsec, and *Real World Haskell*), first released in April
2014[^1]. Its pitch was ergonomics: where the underlying `http-client`
exposes a record-based, ceremonial API, wreq wraps it in lenses so that a
GET-plus-JSON-extraction is a two-liner. It bundles sessions with keep-alive
and cookie persistence, multipart uploads, basic and OAuth2 bearer auth,
automatic decompression, and — unusually for a general-purpose client —
AWS Signature Version 4 request signing[^2].

The defining tradeoff is the `lens` dependency. For teams already using lens,
wreq's API (`r ^. responseBody . key "name" . _String`) is the most pleasant
HTTP interface in the Haskell ecosystem. For everyone else, wreq drags in one
of Hackage's heaviest transitive dependency closures plus an operator
vocabulary (`^.`, `^?`, `.~`, `&`) to learn before the first request.
Competing libraries (`req`, plain `http-client`) exist largely as reactions
to this choice.

The project is in maintenance mode. Originally at `bos/wreq`, the repository
now lives under the `haskell` organization after O'Sullivan stepped back from
open-source maintenance; the version number has stayed on the 0.5.x line
since 2017, and releases since 2018 have been almost exclusively GHC and
dependency compatibility bumps (Aeson 2.0 in 2023, crypto-ecosystem
migrations through 2026)[^3]. It compiles and works, but the README's "Is it
done? No!" and its TODO list have been static for years.

## Getting Started

Add `wreq` to `build-depends`, or for a quick REPL session:

```bash
cabal repl --build-depends "wreq, lens, lens-aeson"
```

```haskell
{-# LANGUAGE OverloadedStrings #-}
import Network.Wreq
import Control.Lens
import Data.Aeson.Lens (key, _String)

main :: IO ()
main = do
  r <- get "https://httpbin.org/get"
  print (r ^. responseStatus . statusCode)          -- 200
  print (r ^? responseBody . key "url" . _String)   -- Just "https://httpbin.org/get"

  -- query params and headers via Options lenses
  let opts = defaults & param "q" .~ ["haskell"]
  r2 <- getWith opts "https://httpbin.org/get"
  print (r2 ^. responseStatus . statusCode)
```

For multiple requests to the same host, use `Network.Wreq.Session` — it
reuses connections and persists cookies across calls:

```haskell
import qualified Network.Wreq.Session as S

main = do
  sess <- S.newSession
  _ <- S.post sess "https://example.com/login" ["user" := ("tom" :: String)]
  r <- S.get  sess "https://example.com/me"    -- sends the login cookie
  print (r ^. responseStatus . statusCode)
```

## Architecture / How It Works

wreq is a thin ergonomic layer over Michael Snoyman's `http-client` /
`http-client-tls` stack[^4], which does the actual connection management,
TLS (via the pure-Haskell `tls` package), and redirect handling.

- **Options + lenses.** `Options` wraps `http-client`'s `Request` plus
  wreq-level settings (auth, proxy, params, response checking). Every field
  is exposed as a lens, so requests are built by threading `defaults`
  through `&` chains. Response accessors (`responseStatus`, `responseBody`,
  `responseHeader`) are likewise lenses over `http-client`'s `Response`.
- **Sessions.** `Network.Wreq.Session` pairs an `http-client` `Manager`
  (connection pool) with a cookie jar held in shared mutable state, giving
  keep-alive and cookie persistence. The top-level convenience functions
  (`get`, `post`, ...) do *not* share a manager between calls — connection
  reuse only happens inside a `Session`.
- **Payload typeclasses.** `Postable` / `Putable` dispatch on the body type:
  form params, multipart `Part`s (file uploads via `http-client`'s multipart
  support), raw bytestrings, and — since 0.5.3.0 — any aeson `ToJSON` value.
- **JSON.** `asJSON` decodes a response into a typed `FromJSON` value;
  schema-less navigation goes through `lens-aeson` (`key`, `nth`, `_String`)
  applied directly to `responseBody`.
- **AWS SigV4.** Signing is implemented in-tree against low-level crypto
  packages, which is why crypto-ecosystem deprecations keep forcing
  maintenance releases[^3].

## Production Notes

- **Non-2xx responses throw by default.** Inherited from `http-client`: a
  404 raises an `HttpException` rather than returning a response. To inspect
  error bodies, override the `checkResponse` lens in `Options` (renamed from
  `checkStatus` in 0.5.0.0, breaking most pre-2017 tutorial code)[^3].
- **No streaming.** `responseBody` is a lazy `ByteString` fully read into
  memory. `foldGet` allows incremental consumption, but for large downloads
  or proxying, `http-client`'s `withResponse` or `http-conduit` are the
  right tools; wreq is not.
- **Use Sessions or pay per-request setup.** Top-level `get`/`post` calls
  spin up fresh connection state, so a naive loop performs a TLS handshake
  per iteration. Code making more than one request should use a `Session`.
- **Compile-time cost.** The `lens` and `aeson` dependencies dominate cold
  build times; on constrained CI this is the main reason teams pick `req` or
  raw `http-client` instead[^5].
- **AWS signing is a convenience, not an SDK.** SigV4 support predates the
  mature `amazonka` ecosystem; for real AWS work use amazonka, which knows
  service endpoints, retries, and pagination.
- **Maintenance-mode hygiene.** Recent releases track GHC and dependency
  churn only; the README still carries a dead Travis CI badge. Expect
  correct behavior but no new features, and check the changelog for
  compatibility bounds before major GHC upgrades[^3].

## When to Use / When Not

**Use when:**
- Your codebase already uses lens and you want the tersest possible HTTP
  calls, especially against JSON APIs.
- You need cookie-persisting sessions, multipart uploads, or basic/bearer
  auth without hand-assembling `http-client` requests.
- You are learning Haskell web clients — the serpentine.com tutorial remains
  one of the better-written in the ecosystem[^1].

**Avoid when:**
- You need streaming request or response bodies — use `http-client` /
  `http-conduit` directly.
- You want to minimize dependency footprint or build time — `req` or plain
  `http-client` are lighter.
- You need type-level guarantees against malformed requests — `req` makes
  invalid requests unrepresentable; `servant-client` derives clients from
  typed API descriptions.
- You expect active feature development from your HTTP layer.

## Alternatives

- snoyberg/http-client — the engine underneath wreq; use it directly when
  you need streaming, fine-grained control, or minimal dependencies.
- mrkkrp/req — type-safe API that statically rejects invalid requests;
  use it when you want correctness guarantees without the lens dependency.
- snoyberg/http-conduit — conduit-based streaming layer over http-client;
  use it for large bodies and constant-memory transfers.
- haskell-servant/servant — servant-client generates clients from type-level
  API specs; use it when you control or can describe the whole API.
- brendanhay/amazonka — full AWS SDK; use it instead of wreq's SigV4 signing
  for anything beyond trivial AWS calls.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0.0 | 2014-04-22 | Initial release by Bryan O'Sullivan[^3]. |
| 0.3.0.0 | 2014-12-02 | AWS request signing, custom HTTP verbs. |
| 0.4.0.0 | 2015-05-10 | GHC 7.10, `withAPISession`, S3 virtual-host URLs. |
| 0.5.0.0 | 2017-01-09 | http-client >= 0.5; `checkStatus` renamed `checkResponse`. |
| 0.5.2.0 | 2018-01-01 | `newSession` replaces deprecated `withSession`. |
| 0.5.3.0 | 2018-11-16 | JSON (`ToJSON`) `Postable`/`Putable` instances. Last feature release. |
| 0.5.4.0 | 2023-03-01 | Aeson 2.0 compatibility, PATCH support; community maintenance. |
| 0.5.4.5 | 2026-02-16 | Dependency migration (memory → ram)[^3]. |

## References

[^1]: Bryan O'Sullivan, "wreq: a Haskell web client library" — tutorial site. http://www.serpentine.com/wreq/
[^2]: wreq on Hackage. https://hackage.haskell.org/package/wreq
[^3]: wreq changelog. https://github.com/haskell/wreq/blob/master/changelog.md
[^4]: http-client on Hackage. https://hackage.haskell.org/package/http-client
[^5]: lens on Hackage. https://hackage.haskell.org/package/lens

## Tags

haskell, http-client, networking, lens, json, api-client, aws-sigv4, library, maintenance-mode
