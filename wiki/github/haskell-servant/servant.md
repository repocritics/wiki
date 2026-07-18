# haskell-servant/servant

> A Haskell DSL that describes web APIs as types — one API definition, many
> interpretations: server, client, docs, mocks.

[GitHub repo](https://github.com/haskell-servant/servant) ·
[Official website](https://docs.servant.dev/) ·
License: BSD-3-Clause (per package; the monorepo root has no detected license)

## Overview

Servant is a type-level web framework for Haskell, first released in December
2014 and formalized in the WGP 2015 paper "Type-level Web APIs with
Servant"[^1]. Its core idea is unusual among web frameworks in any language:
you describe an entire HTTP API as a *type* — built from combinators like
`Capture`, `QueryParam`, `ReqBody`, and `:<|>` — and then derive artifacts
from that single type. `servant-server` interprets it as a WAI application,
`servant-client` derives typed client functions, `servant-openapi3` derives an
OpenAPI spec, `servant-foreign` generates foreign clients, and `servant-docs`
produces documentation. The API type is the single source of truth; the
compiler enforces that server handlers, clients, and docs never drift apart.

The defining tradeoff is compile-time guarantees versus type-level machinery
cost. Servant makes an entire category of bugs (route/handler mismatch,
wrong status codes for content negotiation, stale client bindings)
unrepresentable, but pays for it with slow compiles on large APIs, multi-page
type errors when something does not line up, and a real prerequisite of
intermediate-to-advanced Haskell (`DataKinds`, type families, type classes
with overlapping concerns) to debug or extend it.

At ~2.0k stars servant looks small next to mainstream frameworks, but the
number understates its position: within Haskell it is the default choice for
typed HTTP services, it is used in production at companies running Haskell
backends, and the repo remains actively maintained (pushed within days as of
mid-2026). The 295 open issues span a multi-package monorepo (servant,
servant-server, servant-client, servant-docs and friends), so the raw count
reads higher than the core library's actual defect surface.

## Getting Started

Add to your `.cabal` file: `build-depends: base, servant, servant-server, warp`.

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE TypeOperators #-}
module Main where

import Network.Wai.Handler.Warp (run)
import Servant

-- The API as a type: GET /hello?name=<string> returning JSON
type API = "hello" :> QueryParam "name" String :> Get '[JSON] String

server :: Server API
server = pure . maybe "Hello, world" ("Hello, " ++)

main :: IO ()
main = run 8080 (serve (Proxy :: Proxy API) server)
```

The handler's type is *computed from* the API type: `QueryParam "name" String`
becomes a `Maybe String` argument, `Get '[JSON] String` becomes a
`Handler String` result. Change the API type and the program stops compiling
until every handler, client, and doc generator agrees. The official tutorial
at docs.servant.dev is the canonical on-ramp[^2].

## Architecture / How It Works

The API DSL is a set of uninhabited types used only at the type level.
`:>` chains path segments and request-extracting combinators; `:<|>` joins
alternative endpoints. Interpretation happens through type classes:

- **`HasServer api`** turns an API type plus matching handlers into a WAI
  `Application` (`serve`). Handler types are computed by the associated type
  `ServerT api m`. Under the hood servant builds a static router tree at
  startup rather than matching routes linearly, a design introduced in the
  0.5 internals rework[^3].
- **`HasClient api`** derives client functions (`client`) with the same
  computed types, running over http-client via `servant-client`.
- **`HasDocs` / `HasOpenApi` / `HasForeign`** derive documentation, OpenAPI
  specs, and foreign-language client generators from the same type.

Custom monads are supported through `hoistServer`, which natural-transforms
`ServerT api m` handlers into the concrete `Handler` monad; the `ReaderT`
environment pattern is the standard way to thread app state. `Raw` is the
escape hatch: it embeds any WAI application (static files, websockets,
legacy handlers) at a route, bypassing the type machinery.

Since 0.19, **NamedRoutes** lets APIs be declared as Haskell records with
one field per endpoint instead of anonymous `:<|>` chains[^4]. This fixes
the classic footgun where handlers assembled in the wrong order produced
baffling errors, and gives fields names that show up in type errors.

Writing a *new* combinator means writing `HasServer`/`HasClient`/`HasDocs`
instances by hand — powerful, but it exposes you to the library's internal
`Delayed`/router plumbing and is where the learning curve turns vertical.

## Production Notes

**Compile times scale with API size.** Type-class resolution over large
`:<|>` chains is superlinear in practice. Standard mitigations: split the API
type and its `Server` into per-domain modules, use NamedRoutes, and avoid
re-deriving clients/docs in the same module as the server.

**Type errors are the tax.** A missing `ToJSON` instance or a handler in the
wrong position can produce pages of instance-resolution output ending in an
unhelpful "no instance for HasServer". Teams that succeed with servant learn
to read these errors; newcomers routinely lose hours. NamedRoutes and newer
GHC error messages have improved but not eliminated this.

**Runtime performance is fine.** Servant sits on warp, one of the faster
HTTP servers in any managed language, and the project maintains its own
TechEmpower Framework Benchmarks entry to keep itself honest[^5]. Routing
overhead versus raw WAI is modest; serialization (aeson) usually dominates.

**Ecosystem version skew.** The value proposition depends on companion
packages (servant-auth, servant-openapi3, servant-websockets,
servant-quickcheck), many maintained outside the core repo. After a new GHC
or a core servant release, expect a lag window where some companions have not
bumped bounds yet. Pin your ecosystem with a Stackage snapshot or a
cabal freeze file.

**Auth is an add-on.** Core servant ships a generalized authentication
mechanism that is deliberately minimal; most production apps use
`servant-auth`/`servant-auth-server` (JWT + cookie) or hand-roll against
`AuthProtect`. Budget design time here — it is the least "derived for free"
part of a servant app.

**Streaming works but is awkward.** `StreamGet`/`SourceIO` cover
server-sent streaming; anything more interactive (websockets) drops to
`Raw` + wai-websockets, losing the typed-API benefits at exactly that route.

## When to Use / When Not

**Use when:**
- Your team already writes Haskell and wants the compiler to enforce
  API/handler/client/docs consistency.
- You maintain internal service-to-service APIs where derived typed clients
  eliminate a whole class of integration bugs.
- You need an OpenAPI spec that provably matches the implementation.

**Avoid when:**
- The team is new to Haskell — servant's error messages assume fluency, and
  it is a harsh first Haskell project.
- The service is a small one-off; a Sinatra-style micro-framework ships the
  same thing in a fraction of the code and compile time.
- You need heavy websocket/streaming interactivity as the core workload —
  you would be living in the `Raw` escape hatch anyway.
- You are not on Haskell at all: the pattern exists natively elsewhere
  (tapir on Scala) without cross-language adoption cost.

## Alternatives

- yesodweb/yesod — batteries-included Haskell framework with Template
  Haskell routing; use it when you want a full-stack (forms, auth, persistent
  DB layer) framework rather than an API description language.
- scotty-web/scotty — Sinatra-style minimal Haskell framework; use it for
  small services or prototypes where type-level APIs are overkill.
- digitallyinduced/ihp — Rails-like Haskell framework with its own tooling
  and deployment story; use it for CRUD-heavy product apps built by smaller
  teams.
- softwaremill/tapir — the closest analogue in Scala (typed endpoint
  descriptions, multiple interpreters); use it when your stack is Scala.
- yesodweb/wai — the raw interface servant compiles down to; use it directly
  when you need full control and no DSL.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2 | 2014-12 | First Hackage release; server and client interpretations[^6]. |
| — | 2015-08 | "Type-level Web APIs with Servant" published at WGP 2015[^1]. |
| 0.5 | 2016 | Major internals rework; router tree, restructured packages[^3]. |
| 0.16 | 2019 | `ServantErr` renamed `ServerError`; error-handling cleanup[^3]. |
| 0.19 | 2022 | NamedRoutes: record-based API definitions[^4]. |
| 0.20 | 2023 | Maintenance line; newer GHC support, ecosystem catch-up[^3]. |

## References

[^1]: Alp Mestanogullari, Sönke Hahn, Julian K. Arni, Andres Löh, "Type-level Web APIs with Servant" — WGP 2015. https://www.andres-loeh.de/Servant/servant-wgp.pdf
[^2]: Servant tutorial. https://docs.servant.dev/en/latest/tutorial/index.html
[^3]: servant CHANGELOG. https://github.com/haskell-servant/servant/blob/master/servant/CHANGELOG.md
[^4]: servant 0.19 release notes (NamedRoutes). https://hackage.haskell.org/package/servant-0.19/changelog
[^5]: Servant TechEmpower Framework Benchmarks entry (maintained by the servant team). https://github.com/haskell-servant/FrameworkBenchmarks
[^6]: servant on Hackage. https://hackage.haskell.org/package/servant

## Tags

haskell, web-framework, type-level-programming, rest-api, api-design, code-generation, openapi, dsl, server, http-client
