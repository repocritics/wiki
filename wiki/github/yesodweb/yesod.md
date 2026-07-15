# yesodweb/yesod

> A Haskell web framework that pushes routing, templates, and URL correctness into the type system, checked at compile time.

[GitHub repo](https://github.com/yesodweb/yesod) ·
[Official website](http://www.yesodweb.com/) ·
[License: MIT](https://github.com/yesodweb/yesod/blob/master/LICENSE)

## Overview

Yesod is a RESTful web framework for Haskell built on top of WAI (the Web Application Interface) and typically served by the Warp HTTP server[^1]. It was created by Michael Snoyman, who also authored much of the surrounding ecosystem — WAI/Warp, the Persistent database library, the Shakespearean template family, conduit, and later the Stack build tool. Yesod is the "batteries-included" end of Haskell web development: it ships routing, templating, sessions, authentication scaffolding, forms, and a project generator, in contrast to the assemble-it-yourself approach of lighter WAI frameworks.

Its defining idea is to move correctness checks that most frameworks perform at runtime — does this route exist, does this template reference a valid variable, is this URL well-formed — into the compiler. Routes are generated as an algebraic data type, so a dead internal link is a type error, not a 404 in production. Templates are parsed at compile time via Template Haskell, so a typo in an interpolated variable fails the build. This is Yesod's central tradeoff: strong guarantees and refactoring safety in exchange for heavy reliance on advanced GHC extensions (Template Haskell, QuasiQuotes, type families), long compile times, and error messages that can be opaque to newcomers.

As of 2026 the project is modestly starred (~2,700) but steadily maintained, with commits landing within the last week[^2]. It occupies a stable, mature niche rather than a growth curve; the Haskell web audience is small, and much of the framework's surface has been stable since the 1.x line.

## Getting Started

Yesod is normally used through the Stack build tool with a scaffolded project template[^3]:

```bash
stack new my-project yesod-sqlite && cd my-project
stack build
stack exec -- yesod devel   # auto-recompiling dev server
```

A single-file application, showing the Template Haskell core:

```haskell
{-# LANGUAGE OverloadedStrings, QuasiQuotes, TemplateHaskell, TypeFamilies #-}
import Yesod

data App = App   -- the "foundation" type: config, DB pool, HTTP manager, etc.

mkYesod "App" [parseRoutes|
/          HomeR   GET
/users/#UserId UserR GET
|]

instance Yesod App

getHomeR :: Handler Html
getHomeR = defaultLayout [whamlet|<a href=@{UserR 1}>First user|]

getUserR :: UserId -> Handler Html
getUserR uid = defaultLayout [whamlet|User ##{show uid}|]

main :: IO ()
main = warp 3000 App
```

The `@{UserR 1}` interpolation is a type-safe URL: if the `UserR` route is renamed or its parameter type changes, this line fails to compile.

## Architecture / How It Works

Yesod sits on a layered stack. **WAI** is the minimal common interface between web applications and servers — a request-to-response function. **Warp** is the production WAI server. Yesod builds application logic above WAI, and any WAI middleware (gzip, logging, CORS) composes underneath it.

The framework's mechanics center on a few Template Haskell entry points:

- **`parseRoutes` + `mkYesod`** — a route DSL is parsed at compile time into a URL data type plus dispatch code. Each resource names a handler function per HTTP method (`getHomeR`, `postUserR`). Missing handlers are compile errors.
- **The foundation type** — your `App` value holds shared state (DB connection pool, settings, logger). Typeclass instances (`Yesod`, `YesodPersist`, `YesodAuth`) are where you override behavior like the default layout, authorization rules, and session backend.
- **The `Handler` monad** — where request handling runs: short-circuiting responses, session access, subsite dispatch, and database queries all live here.
- **Widgets** — the `Widget` type is a monoid that accumulates HTML, CSS, and JS fragments plus their `<head>` requirements. Composing widgets automatically deduplicates and orders included assets.
- **Shakespearean templates** — Hamlet (HTML), Lucius/Cassius (CSS), and Julius (JS) are compile-time-checked template languages with typed interpolation: `#{expr}` for values, `@{route}` for URLs, `^{widget}` for embedding[^4].

Two major companion libraries are technically separate repos but part of the same design: **Persistent**, a type-safe datastore layer with backends for PostgreSQL, SQLite, MySQL, and MongoDB; and **Shakespeare**, the template family. The `yesod` package on Hackage is a meta-package pulling together `yesod-core`, `yesod-form`, `yesod-auth`, `yesod-persistent`, and `yesod-static`.

The whole design leans on GHC's type system as the enforcement mechanism, which is why a Yesod app rebuilds slowly but rarely surprises you at runtime.

## Production Notes

**Compile times and Template Haskell.** The type safety is paid for at build time. Template Haskell splices force recompilation of dependent modules and defeat some incremental-build caching; large route tables and many templates make full rebuilds noticeably slow. `yesod devel` mitigates the edit-reload loop but does not remove the underlying cost. Keeping handlers in separate modules from the route/foundation definitions limits recompilation blast radius.

**Error messages.** Failures inside `parseRoutes`, `whamlet`, or Persistent's `mkPersist` quasi-quotes surface as Template Haskell errors that often point at the splice site rather than the real mistake. This is the steepest part of the newcomer learning curve — the guarantees are real, but diagnosing why the build broke takes practice.

**Deployment.** Output is a single statically-inclined native binary (plus static assets and, if used, a database). There is no interpreter or JVM to provision. The practical friction is the build environment, not the runtime: GHC + the dependency set is large, so CI build caching (Stack's or cabal's store) matters a lot. The binary itself is memory-stable and fast; Warp is a genuinely high-throughput server.

**Ecosystem size.** The blessing and curse of Haskell. Core libraries (WAI, Warp, Persistent, aeson) are solid and well-maintained, but for a given third-party integration you may find no library, an unmaintained one, or one you must bind to C yourself. Fewer Stack Overflow answers exist than for mainstream frameworks; the Yesod book[^1] and library Haddocks are the primary references.

**Version stability.** The 1.6 line has held for years; breaking changes across the framework are infrequent and usually driven by GHC or conduit major bumps rather than Yesod redesigns. Upgrades are more often about the wider dependency snapshot (an LTS Stackage bump) than Yesod itself.

## When to Use / When Not

**Use when:**
- Your team already writes Haskell and wants compile-time guarantees to extend to routing, templates, and URLs.
- Correctness and refactoring safety matter more than iteration speed — internal tools, correctness-critical services.
- You want an integrated stack (routing + forms + auth + typed DB via Persistent) rather than wiring pieces together.

**Avoid when:**
- The team doesn't know Haskell; the framework is not the place to learn it, and hiring is hard.
- You need rapid prototyping with fast feedback loops — compile times and TH errors work against you.
- You depend on a large plugin/integration ecosystem or lots of ready-made SaaS SDKs.
- You want a small, transparent codebase you can read top-to-bottom — the Template Haskell machinery is the opposite of that.

## Alternatives

- haskell-servant/servant — type-level API DSL; APIs are described as types, giving free client/docs generation. More composable and library-like, less batteries-included; common for typed JSON APIs.
- digitallyinduced/ihp — Haskell framework with a Rails-like, convention-heavy developer experience and gentler onboarding; use when you want Haskell but faster ramp-up.
- scotty-web/scotty — minimal Sinatra-style layer over WAI/Warp; use for small services where Yesod's machinery is overkill.
- agrafix/Spock — lightweight, stateful WAI framework with typed routing but far less scaffolding than Yesod.
- Non-Haskell teams wanting similar compile-time guarantees typically reach for Rust (actix-web, axum) or a typed-first stack instead.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2009 | Repository created; early WAI-based experiments[^2]. |
| 1.0 | 2012 | First stable major release; API consolidation[^1]. |
| 1.2 | 2013 | Subsite system rework; conduit-based streaming. |
| 1.4 | 2014 | Ecosystem stabilization across yesod-* packages. |
| 1.6 | 2018 | Conduit 1.3 / newer GHC support; long-lived current line. |

## References

[^1]: Michael Snoyman, "Developing Web Applications with Haskell and Yesod" (the Yesod book). https://www.yesodweb.com/book
[^2]: GitHub repository metadata, yesodweb/yesod — created 2009-06-27, ~2,720 stars, MIT license, last push 2026-07-09. https://github.com/yesodweb/yesod
[^3]: Yesod quick start guide (Stack-based scaffolding). http://www.yesodweb.com/page/quickstart
[^4]: Yesod book, "Shakespearean Templates". https://www.yesodweb.com/book/shakespearean-templates

## Tags

haskell, web-framework, wai, warp, template-haskell, type-safe, server-side-rendering, restful, persistent, backend
