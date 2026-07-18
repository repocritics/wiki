# kemalcr/kemal

> Sinatra-style web framework for Crystal — compiled-language throughput with a
> scripting-language DSL, in a deliberately small package.

[GitHub repo](https://github.com/kemalcr/kemal) ·
[Official website](https://kemalcr.com) ·
[License: MIT](https://github.com/kemalcr/kemal/blob/master/LICENSE)

## Overview

Kemal is the de facto default web framework for the Crystal language, created
by Serdar Doğruyol in 2015 and descended from Manas' earlier Crystal framework
Frank[^1]. It ports the Sinatra idiom — top-level `get "/" do ... end` route
blocks — to a compiled, statically typed language, so the resulting binary
serves HTTP with low latency and a small memory footprint while the source
reads like a Ruby microframework. It reached 1.0.0 on 2021-03-22, the same day
Crystal itself hit 1.0[^2], and has tracked Crystal releases since.

The defining tradeoff is inherited from Crystal, not chosen by Kemal: you get
native-code performance and compile-time type checking, but you buy into a
niche ecosystem. At ~3.9k stars Kemal is one of the largest Crystal projects,
yet the surrounding shard ecosystem (ORMs, auth, background jobs) is a
fraction of what Ruby or Node offers, and hiring for Crystal is hard. Kemal
itself stays intentionally thin: routing, middleware, WebSockets, static
files, and ECR templates in core; sessions, CSRF, and auth live in separate
shards under the kemalcr org[^3]. Maintenance is steady rather than fast —
a handful of releases per year, driven largely by Crystal version compatibility,
with the last push in July 2026 and only 4 open issues.

## Getting Started

Add Kemal to a Crystal project's `shard.yml`:

```yaml
dependencies:
  kemal:
    github: kemalcr/kemal
```

```crystal
require "kemal"

get "/" do
  "Hello World!"
end

get "/api/status" do |env|
  env.response.content_type = "application/json"
  {"status": "ok"}.to_json
end

ws "/chat" do |socket|
  socket.send "Hello from Kemal WebSocket!"
end

Kemal.run
```

```bash
shards install
crystal run src/my_app.cr        # dev
crystal build --release src/my_app.cr   # production binary
```

## Architecture / How It Works

Kemal is a thin layer over Crystal's standard-library `HTTP::Server`. Almost
everything is an `HTTP::Handler` in a middleware chain: an init handler,
logger, exception handler, static-file handler (derived from stdlib's),
WebSocket handler, and finally the route handler[^4]. `add_handler` inserts
user middleware into this chain; writing custom middleware means subclassing
`Kemal::Handler`, the same shape as stdlib middleware.

Routes declared with the `get`/`post`/`put`/`patch`/`delete`/`ws` macros are
registered at program start into a global route handler and matched with a
radix tree, which keeps lookup cost roughly independent of route count[^4].
Each route block receives `env` (`HTTP::Server::Context`) with a unified
params API: `env.params.url` (path segments), `env.params.query`,
`env.params.body`, and `env.params.json`. Templates use ECR, Crystal's
compile-time embedded template format — templates are compiled into the
binary, so a template typo is a build error, not a 500.

Concurrency is Crystal's model: a single-threaded event loop scheduling
lightweight fibers, with non-blocking IO in the stdlib. A Kemal process
handles many concurrent connections on one core; multi-core execution
requires Crystal's still-flagged multithreading (`-Dpreview_mt`) or running
multiple processes. Kemal exposes `reuse_port` in its config so several
processes can bind the same port behind the kernel's SO_REUSEPORT balancing.

The design leans on global state: routes, config (`Kemal.config`), and the
handler chain are process-global singletons. That is what makes the four-line
hello world possible, and also what makes embedding two Kemal apps in one
binary impractical.

## Production Notes

- **One process = one core.** Without `-Dpreview_mt`, CPU-bound work blocks
  the fiber scheduler and every in-flight request behind it. Standard
  deployment is N processes with `reuse_port: true` or a reverse proxy.
  Blocking C library calls (some DB drivers) stall the event loop the same way.
- **Compile-before-deploy.** `--release` builds of nontrivial apps take
  noticeably longer than dev builds; CI pipelines should cache. The upside is
  a single static-linkable binary — Alpine/musl containers with no runtime
  are the normal shipping artifact.
- **Crystal version coupling.** Most Kemal releases exist to track Crystal
  compiler releases. Upgrading Crystal before your shards (or Kemal) support
  it is the most common breakage; pin both. Pre-1.0 Crystal broke source
  compatibility routinely; post-1.0 this pain is largely historical.
- **Batteries are elsewhere.** Sessions (`kemal-session`), CSRF
  (`kemal-csrf`), and auth middleware are separate shards with their own
  maintenance cadence — audit them independently; some community shards for
  Crystal go dormant. Database access goes through `crystal-db` drivers.
- **Testing** uses the `spec-kemal` shard, which exercises the app in-process.
  Because routes are global, test isolation between multiple "apps" in one
  spec suite requires care.
- **Windows** is a second-class target inherited from Crystal's still-maturing
  Windows support; production Kemal runs on Linux (and macOS for dev).

## When to Use / When Not

**Use when:**
- You want Sinatra/Express ergonomics but need the throughput and memory
  profile of a compiled language on modest hardware.
- You are building small-to-medium JSON APIs, webhooks, or WebSocket services
  where a single static binary simplifies deployment.
- You already chose Crystal and want the most battle-tested HTTP layer in
  that ecosystem.

**Avoid when:**
- You need a deep library ecosystem (payments SDKs, enterprise SSO, mature
  ORMs) — Ruby, Node, or Go will cost less in glue code.
- You need easy parallel CPU work per process; Crystal's multithreading is
  still behind a preview flag.
- Your team cannot absorb a niche language's hiring and onboarding cost.
- You want full-stack conventions (ORM, migrations, asset pipeline) out of
  the box — Kemal deliberately does not provide them.

## Alternatives

- luckyframework/lucky — use instead for a full-stack, strongly conventioned
  Crystal framework with compile-time-checked routes and an ORM (Avram).
- athena-framework/athena — use instead if you prefer annotation-driven,
  structured (Symfony-flavored) architecture over a DSL in Crystal.
- amberframework/amber — Rails-like Crystal MVC; consider only with a
  maintenance-status check, as activity has been intermittent.
- sinatra/sinatra — the Ruby original; use instead when ecosystem breadth
  matters more than raw performance.
- gin-gonic/gin — use instead for a similar minimal-router shape in Go, with
  a much larger ecosystem and easier hiring.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.11 era | 2015-10 → 2016 | Repo created 2015-10-23; rapid pre-1.0 iteration tracking pre-1.0 Crystal[^5]. |
| v0.16.0 | 2016-10-01 | Mid pre-1.0 series; API largely settled into its current DSL shape. |
| v1.0.0 | 2021-03-22 | Stable release, same day as Crystal 1.0[^2]. |
| v1.3.0 | 2022-10-09 | Crystal 1.x compatibility line continues. |
| v1.6.0 | 2024-10-12 | Maintenance cadence of roughly 1–2 minor releases/year. |
| v1.7–v1.9 | 2025 | Three minor lines in one year — uptick in activity[^5]. |
| v1.11.0 | 2026-04-13 | Latest release as of this page[^5]. |

## References

[^1]: Kemal README, Acknowledgments — "Special thanks to Manas for their work
    on Frank." https://github.com/kemalcr/kemal#acknowledgments
[^2]: Kemal v1.0.0 release (2021-03-22)
    https://github.com/kemalcr/kemal/releases/tag/v1.0.0 ; Crystal 1.0
    announcement https://crystal-lang.org/2021/03/22/crystal-1.0-what-to-expect/
[^3]: kemalcr GitHub organization (kemal-session and companion shards).
    https://github.com/kemalcr
[^4]: Kemal source: middleware handler chain and route handler.
    https://github.com/kemalcr/kemal/tree/master/src/kemal
[^5]: Kemal releases. https://github.com/kemalcr/kemal/releases

## Tags

crystal, web-framework, microframework, sinatra-like, rest-api, websocket, json-api, middleware, compiled, low-latency
