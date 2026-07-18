# amberframework/amber

> A Rails-style MVC framework for Crystal — batteries included, compiled to a
> native binary, maintained at a slow burn.

[GitHub repo](https://github.com/amberframework/amber) ·
[Official website](https://amberframework.org) ·
[License: MIT](https://github.com/amberframework/amber/blob/master/LICENSE)

## Overview

Amber is a full-stack web framework for [Crystal](https://crystal-lang.org/), a
compiled, statically typed language with Ruby-like syntax. Started in April
2017, drawing on Kemal, Rails, Phoenix, and Hanami[^1], it transplants the
Rails model — MVC, generators, scaffolding, migrations, convention over
configuration — onto a language that compiles to one native binary. The pitch
is Ruby ergonomics with C-class speed plus compile-time type checking; on raw
throughput it holds — Crystal frameworks place well above Rails in the
TechEmpower benchmarks the README still cites (Round 18, 2019)[^2].

The defining tension is ecosystem gravity. Amber peaked with the 2017–2018
Crystal hype wave; at ~2,600 stars it remains one of the two best-known Crystal
web frameworks (alongside Lucky), but the last tagged release is v1.4.1 from
August 2023[^3]. The repository is not dead — commits landed as recently as
June 2026, mostly Crystal-version compatibility work — but with no release in
roughly three years it is best described as maintenance-mode: usable, stable in
scope, unlikely to grow new features. Adopting it means adopting Crystal's
small ecosystem plus a framework whose core contributors have largely moved on.

## Getting Started

```bash
brew install amber          # macOS/Linux; on other Linux, build from source
amber new myblog
cd myblog
shards install
amber watch                 # compile + run, rebuild on file changes
```

Routing uses Phoenix-style pipelines of middleware "pipes":

```crystal
# config/routes.cr
Amber::Server.configure do
  pipeline :web do
    plug Amber::Pipe::Logger.new
    plug Amber::Pipe::Session.new
    plug Amber::Pipe::CSRF.new
  end

  routes :web do
    get "/", HomeController, :index
    resources "/posts", PostsController
  end
end
```

Controllers are plain classes; `amber generate scaffold Post title:string`
produces model, controller, views, and a migration in one step.

## Architecture / How It Works

Amber is a conventional MVC stack whose one structural novelty is that it is
resolved at compile time: routes, dispatch, and templates are Crystal code
compiled into the binary — no runtime route table to misconfigure, and a typo
in a controller action name is a compile error, not a 500.

- **Router + pipelines** — requests flow through named pipelines (`:web`,
  `:api`) of composable pipes (logger, session, CSRF, CORS), a design lifted
  from Phoenix's plug system.
- **Controllers/views** — controllers render ECR (embedded Crystal, analogous
  to ERB) or Slang templates; templates compile with the app, so undefined
  variables fail the build.
- **ORM** — scaffolding targets Granite[^4], the ORM that grew up inside the
  Amber organization (PostgreSQL/MySQL/SQLite adapters); it is a separate
  shard, swappable for Jennifer or raw crystal-db.
- **CLI** — `amber` handles generators, database commands, migrations, an
  encrypted-secrets workflow, and the `watch` rebuild loop.
- **WebSockets** — first-class channel/socket support on Crystal's fiber-based
  HTTP server.

Concurrency is inherited from Crystal: cooperative fibers on a single thread
by default (multi-threading sits behind Crystal's `preview_mt` flag) — Node-
like event-loop throughput without callbacks, but a blocking C library call
blocks every request on that thread.

## Production Notes

- **Compile times are the development tax.** Every change triggers a full
  recompile; small apps rebuild in seconds, larger ones in minutes — a real
  regression from the Rails reload-and-refresh loop Amber otherwise imitates.
- **Release stagnation is the operational risk.** v1.4.1 (2023-08) predates
  several Crystal releases; you may need to pin Amber to `master` in
  `shard.yml` rather than a tag, making builds less reproducible[^3].
- **Deployment is genuinely simple.** One native binary, small memory
  footprint (tens of MB where Rails takes hundreds); you still need matching
  shared libs (libssl, libyaml, libpcre, DB clients) unless you static-link
  against musl.
- **Ecosystem shallowness bites late.** Shards cover the basics, but
  background jobs, admin panels, uploads, and auth have one or zero maintained
  options each; expect to write glue Rails or Phoenix users get for free.
  Hiring Crystal developers is similarly hard.
- **Compile-time safety cuts both ways.** Nil errors and template typos are
  caught before deploy; macro-heavy code yields long, unfriendly errors.
- **Tooling assumes Unix.** Install paths (Homebrew, make-from-source, AUR)
  are Unix-only in practice; Crystal's Windows support arrived late.

## When to Use / When Not

**Use when:**
- You want Rails-shaped structure with an order-of-magnitude less memory and
  latency, and the app's scope is well understood.
- You are already committed to Crystal and want full-stack conventions rather
  than assembling Kemal + ORM + templates yourself.
- Single-binary deployment to modest infrastructure is a real requirement.

**Avoid when:**
- You need an actively evolving framework with a security-response cadence:
  the three-year release gap disqualifies Amber for compliance-sensitive work.
- Your team values ecosystem depth (gems/packages, hiring, Stack Overflow
  density) over runtime performance.
- You need Windows-native development or heavy multi-threaded CPU work.
- You are choosing Crystal for type safety above all — Lucky pushes
  compile-time guarantees (typed routes, typed queries) further than Amber.

## Alternatives

- kemalcr/kemal — Sinatra-class Crystal micro-framework; use it when you want
  minimal routing and will pick your own libraries.
- luckyframework/lucky — the other full-stack Crystal framework; use it for
  stricter compile-time checking of routes/queries and a more active core team.
- athena-framework/athena — annotation-driven, Symfony-flavored Crystal
  framework; use it if you prefer explicit DI over Rails conventions.
- rails/rails — use it when developer ecosystem and hiring matter more than
  runtime cost.
- phoenixframework/phoenix — use it for the same pipeline/channel ideas with a
  mature ecosystem and battle-tested concurrency (BEAM).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2017-08 | First public releases; rapid pre-1.0 iteration through 2018[^3]. |
| 0.28–0.36 | 2019–2021 | Cadence slows; releases largely track Crystal compiler updates. |
| 1.0.0rc1/rc2 | 2021-06 | 1.0 release candidates, following Crystal 1.0 (2021-03)[^5]. |
| 1.2.1 | 2021-12 | First stable 1.x on the GitHub releases page. |
| 1.3.0 | 2022-10 | Crystal compatibility and fixes. |
| 1.4.1 | 2023-08 | Latest tagged release as of mid-2026[^3]. |

## References

[^1]: Amber README and documentation — cites Kemal, Rails, Phoenix, and Hanami as inspirations. https://docs.amberframework.org/amber
[^2]: TechEmpower Framework Benchmarks, Round 18 (2019-07-09), linked from the Amber README. https://www.techempower.com/benchmarks/#section=data-r18
[^3]: Amber releases on GitHub. https://github.com/amberframework/amber/releases
[^4]: Granite ORM, maintained under the Amber organization. https://github.com/amberframework/granite
[^5]: Crystal 1.0 announcement — 2021-03-22. https://crystal-lang.org/2021/03/22/crystal-1.0-what-to-expect/

## Tags

crystal, web-framework, mvc, full-stack, rails-like, scaffolding, orm, compiled, server-side, websockets
