# luckyframework/lucky

> A Rails-shaped web framework for Crystal that moves whole bug classes —
> broken links, nil rendering, mistyped columns — from runtime to compile time.

[GitHub repo](https://github.com/luckyframework/lucky) ·
[Official website](https://luckyframework.org) ·
[License: MIT](https://github.com/luckyframework/lucky/blob/main/LICENSE)

## Overview

Lucky is a full-stack web framework for the Crystal language, started by Paul
Smith in early 2017[^1]. It borrows Rails' convention-over-configuration
posture and Phoenix's single-purpose action classes[^2], then applies
Crystal's static type system to the whole request cycle: routes, query
columns, page inputs, and even HTML rendering are checked at compile time —
a page that prints a nilable column without a fallback does not compile.

The defining tradeoff: this safety comes from replacing templates with code.
HTML is written as Crystal classes and methods (`ul`, `li`, `link`) rather
than ERB-style template files, and pages declare typed inputs with `needs`.
Refactoring is safe and mechanical; the cost is that designers cannot touch
markup without writing Crystal, and every view change is a recompile.

Community scale matters here: ~2.7k stars and a small core team after nearly
a decade. Lucky reached 1.0 in June 2023[^3] after six years of breaking 0.x
releases and remains actively maintained (pushes within days as of mid-2026),
but it is a niche framework in a niche language — you are betting on a small,
dedicated ecosystem, not a mass one.

## Getting Started

Lucky requires Crystal, PostgreSQL, and the `lucky` CLI (macOS/Linux; Windows
via WSL)[^4]:

```bash
brew install luckyframework/homebrew-lucky/lucky   # macOS; see guides for Linux
lucky init        # wizard: app name, full app vs API-only, auth generator
cd my_app
script/setup      # shards install, DB create/migrate
lucky dev         # file-watching dev server
```

A JSON endpoint and model, from the README[^5]:

```crystal
class Api::Users::Show < ApiAction
  get "/api/users/:user_id" do
    user = UserQuery.find(user_id)   # user_id method generated from the route
    json UserSerializer.new(user)
  end
end

class User < BaseModel
  table do
    column last_name : String
    column nickname : String?        # nilable — callers must handle nil
  end
end
```

Mistyping a column name or forgetting a route param is a compile error, not a
500.

## Architecture / How It Works

Lucky is an umbrella over a dozen focused shards, all under the
`luckyframework` org:

- **Actions** — one class per route (`Users::Index < BrowserAction`). Route
  macros generate typed methods for path params; links reference action
  classes (`Users::Show.with(user.id)`) instead of string path helpers, so a
  removed route breaks every caller at compile time.
- **Avram**[^6] — the ORM, PostgreSQL only. `SaveOperation` classes (inspired
  by Elixir's Ecto changesets[^5]) separate validation/persistence from
  models, avoiding Rails-style callback soup. Query objects get
  compile-time-checked, type-specific methods per column (`.lower` on
  strings, `.gt` on times).
- **Pages** — HTML as Crystal method calls; `needs` declares typed inputs.
  Tags auto-close; printing `nil` is a compile error.
- **Supporting shards** — LuckyRouter, Habitat (typed config), Carbon
  (email), Authentic (auth), LuckyFlow (browser testing), Breeze (dev
  dashboard).

The coupling story: Avram is technically separable but generators, guides,
and operations all assume it, and Avram assumes Postgres. Session/cookie/
flash handlers were originally adapted from Amber[^5]. Everything compiles to
a single binary via Crystal's LLVM backend — there is no interpreter or hot
reload; the `lucky watch` dev loop recompiles on change.

## Production Notes

- **Compile times are the tax for the type safety.** Views are code, so every
  template tweak triggers a Crystal recompile; iteration on large apps is
  slower than interpreted-framework hot reload. Compilation is also
  memory-hungry — a 512MB–1GB VM can OOM, so build in CI and ship the binary.
- **PostgreSQL or nothing.** Avram has no MySQL/SQLite support.
- **Deployment is a single binary** — no runtime/bundler on the host — but
  you own cross-compilation or a build-stage Dockerfile; musl/Alpine static
  builds are the common pattern.
- **0.x upgrades were rough** — breaking changes nearly every release for six
  years. Post-1.0 is calmer, but CLI, framework, and Avram versions move in
  lockstep; upgrade them together.
- **Ecosystem depth is the real constraint.** Crystal's shard ecosystem is a
  small fraction of rubygems/npm; expect to hand-write clients for
  less-common SaaS APIs. Hiring Crystal developers is hard; most teams train
  Rubyists (syntax transfers; type-system habits do not).
- **Bus factor.** The original creator has stepped back; a small core team
  maintains the org[^5]. Activity is steady but a fraction of Rails/Phoenix
  volume — check issue response times against your support expectations.

## When to Use / When Not

**Use when:**
- You like Rails' shape but want compile-time guarantees and single-binary
  deploys with low memory footprint.
- Your team refactors constantly and wants the compiler to find every broken
  link, param, and query.
- Postgres is a given and the app is a conventional server-rendered site or
  JSON API.

**Avoid when:**
- Designers or non-Crystal contributors need to edit markup — the HTML-as-code
  DSL is a hard gate.
- You need mature libraries for payments, search, queues, etc. out of the box.
- Hiring pool size or ecosystem risk dominates your framework choice.
- You need a non-Postgres database.

## Alternatives

- rails/rails — use instead when ecosystem depth and hiring matter more than
  type safety or memory footprint.
- phoenixframework/phoenix — same "compiled, fast, productive" pitch with a
  far larger community and a real-time (LiveView) story.
- amberframework/amber — the other batteries-included Crystal framework;
  conventional MVC with template files instead of HTML-as-code.
- kemalcr/kemal — Sinatra-style Crystal micro-framework; use for Crystal's
  speed without Lucky's structure or compile-time ceremony.
- athena-framework/athena — annotation-driven, Symfony-flavored Crystal
  framework for those preferring explicit DI over conventions.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-0.1 | 2017-01 | Repo created by Paul Smith; public 0.x development begins[^1]. |
| 0.x series | 2017–2023 | Breaking changes most releases; Avram split out as the ORM. |
| 1.0.0 | 2023-06 | First stable release; API stability commitment[^3]. |
| 1.x series | 2023–present | Incremental releases tracking Crystal versions; active pushes as of 2026-07. |

## References

[^1]: GitHub repository metadata — created 2017-01-06. https://github.com/luckyframework/lucky
[^2]: Lucky guides, "Why Lucky?". https://luckyframework.org/guides/getting-started/why-use-lucky
[^3]: Lucky v1.0.0 release. https://github.com/luckyframework/lucky/releases/tag/v1.0.0
[^4]: Lucky guides, "Installing Lucky". https://luckyframework.org/guides/getting-started/installing
[^5]: Lucky README (examples, Amber/Ecto attributions, contributor note). https://github.com/luckyframework/lucky#readme
[^6]: Avram — Lucky's PostgreSQL ORM. https://github.com/luckyframework/avram

## Tags

crystal, web-framework, full-stack, mvc, type-safety, compile-time-checks, postgresql, orm, server-side-rendering, rails-alternative
