# ruby-grape/grape

> An opinionated DSL for building REST-like APIs in Ruby — Rack-native, framework-agnostic, and older than most of its alternatives.

[GitHub repo](https://github.com/ruby-grape/grape) ·
[Official website](http://www.ruby-grape.org) ·
[License: MIT](https://github.com/ruby-grape/grape/blob/master/LICENSE)

## Overview

Grape is a micro-framework for writing HTTP APIs in Ruby. It is not a full web framework: it has no ORM, no view layer, and no asset story. What it provides is a declarative DSL — `resource`, `params`, `version`, `desc` — that compiles into plain Rack applications, so a Grape API can run standalone under rackup or be mounted inside Rails or Sinatra[^1]. It was started by Michael Bleigh at Intridea in August 2010, which makes it one of the oldest continuously maintained Ruby projects outside Rails itself; the current stable line is 3.3.x (3.3.2, July 2026), with 4.0 in development requiring Ruby 3.3+[^2].

The defining tradeoff is DSL density versus transparency. A `params` block gives you type coercion, validation, and (via grape-swagger) generated OpenAPI docs from a single declaration — that is the pitch. The cost is a heavily metaprogrammed core: `Grape::API` is not the class you subclass in any ordinary sense, routes are compiled lazily on first request, and settings like `before`/`rescue_from` are inherited across `mount` boundaries in ways you have to learn rather than read. Teams either internalize the DSL and are productive, or fight it and conclude Rails API mode plus strong params would have been simpler.

At roughly 10k stars and 1.2k forks over 16 years, Grape is mature rather than fashionable. Maintenance is genuinely active — pushes within days of this writing, near-monthly releases through 2025–2026 — but the recent changelog is dominated by a small number of maintainers, which is a real bus-factor consideration for a dependency this load-bearing[^2].

## Getting Started

Ruby 3.3+ is required on the upcoming 4.0 line (3.x supports earlier 3.x Rubies)[^1].

```bash
bundle add grape
```

```ruby
# config.ru
require 'grape'

class API < Grape::API
  format :json
  prefix :api
  version 'v1', using: :path

  resource :statuses do
    desc 'Return a status.'
    params do
      requires :id, type: Integer, desc: 'Status ID.'
    end
    route_param :id do
      get do
        { id: params[:id], text: 'hello' }
      end
    end
  end
end

run API
```

`rackup` serves `GET /api/v1/statuses/:id`; a non-integer `:id` is rejected with a 400 before your block runs. Inside Rails, place classes under `app/api` and `mount API => '/'` in `config/routes.rb` — and note that Zeitwerk inflects `api` as `Api`, so you must register `API` as an acronym or the constant will not load[^1].

## Architecture / How It Works

Subclassing `Grape::API` triggers a metaprogramming chain: the DSL methods (`get`, `params`, `namespace`, `helpers`) accumulate settings on an inheritable stack rather than defining methods directly. Each route becomes a `Grape::Endpoint` — itself a small Rack app wrapped in its own middleware stack (error handling, formatter, parser, versioner) — and a `Grape::Router` matches incoming paths using mustermann patterns. Routes compile lazily on the first request; `API.compile!` forces this at boot, which is what you want in production to avoid a slow, thread-sensitive first hit[^1]. In 4.0 the mustermann-grape extension is being inlined into Grape's own router[^2].

The `params` DSL is the framework's center of gravity. Declared parameters are coerced (dry-types has done the coercion since 1.3.0, replacing Virtus[^2]), validated (`requires`, `optional`, `values`, `regexp`, mutually-exclusive groups), and exposed as an `ActiveSupport::HashWithIndifferentAccess` by default (configurable to plain `Hash` or `Hashie::Mash`). Since 2.1.0 there is also a `contract` DSL for validating with dry-schema/dry-validation contracts as an escape hatch from the built-in validators[^2].

Composition happens through `mount`: APIs mount other APIs, and mounted endpoints inherit `before`/`after`/`rescue_from` from their host regardless of declaration order. "Remounting" lets one API class be mounted in several places with per-mount `configuration` hashes — effectively parameterized API templates[^1]. Versioning supports four strategies (`:path`, `:header`, `:accept_version_header`, `:param`), with content-negotiation quality handling on the header strategies.

Presentation and documentation live in sibling gems, not core: grape-entity for serialization[^3] and grape-swagger for OpenAPI generation from `desc` blocks[^4]. Grape's runtime dependencies are deliberately short — rack, activesupport, zeitwerk, dry-configurable, mustermann — but the ActiveSupport dependency means "Rails-free" Grape still carries a chunk of Rails.

## Production Notes

**Call `compile!` at boot.** Lazy route compilation means the first request pays the full route-building cost. On large APIs (hundreds of endpoints) this is noticeable, and under threaded servers it happens under load[^1].

**Validation cost scales with payload shape.** The params DSL validates every element of nested arrays of hashes. Endpoints accepting large batch payloads with deep `params` declarations spend real CPU in validation; the `contract` DSL or hand validation is the usual workaround for hot endpoints.

**`Rack::Cascade` ordering footgun.** When cascading Grape behind Sinatra or Rails, Grape must be last: a 404/405 from a non-final app is treated as "try the next app," which can surface the wrong application's 404 page[^5].

**Rack 3 migration.** Grape 2.0 (2023-11) added Rack 3 support, but the transition had sharp edges: downcased response headers, `Rack::Auth::Digest` removal, and a `Rack::ETag` linting bug under Rails' default middleware stack that was only fixed in Rack 3.1.13/3.0.15[^1][^2]. If you run `lint!`, pin Rack accordingly.

**`return` inside endpoints is deprecated** as of 3.0.0 — endpoint blocks are instance-exec'd, and `return` never behaved like a normal method return; use the block's last expression or `error!`[^2].

**Mount inheritance surprises.** `rescue_from :all` and `before` filters declared on a parent apply to everything mounted below, including APIs written by other teams. This is convenient until an inherited rescue handler masks an error class a sub-API wanted to handle itself. Audit the full mount tree when debugging error handling.

**Observability.** Grape instruments `endpoint_run.grape`, `endpoint_render.grape`, and filter events through ActiveSupport::Notifications; most APM vendors (Datadog, New Relic, Skylight) subscribe to these out of the box.

## When to Use / When Not

**Use when:**
- You want a documented, versioned JSON API with validation and OpenAPI output driven from one DSL, without adopting all of Rails.
- You are adding an API surface to an existing Rails or Sinatra app and want it isolated from the host's routing and controller stack.
- You value a 15-year track record and slow, deliberate API evolution over novelty.

**Avoid when:**
- You are already deep in Rails conventions — Rails API mode with jbuilder/serializers duplicates most of Grape with less concept count.
- Raw throughput is the constraint: Grape's per-request middleware and validation layers are not free, and lighter Rack routers (Roda) or newer async frameworks measure faster.
- Your team dislikes DSL-heavy metaprogramming; Grape stack traces route through instance-exec'd blocks and compiled endpoints, which makes debugging less direct than plain classes.
- You need GraphQL or RPC-style contracts — Grape is REST-shaped by design.

## Alternatives

- rails/rails — use API-only mode instead when you already run Rails and want one framework, one convention set, and ActionController features (caching, params, instrumentation) without a second DSL.
- sinatra/sinatra — use instead when you want minimal routing with no opinions about params, versioning, or documentation.
- jeremyevans/roda — use instead when per-request performance and a small, explicit routing tree matter more than declarative validation and doc generation.
- hanami/hanami — use instead when you want a full-featured, non-Rails framework with a stronger architecture story (containers, slices) rather than an API bolt-on.
- rage-rb/rage — use instead when you want a Rails-compatible API framework built around fiber-based concurrency for high-throughput IO-bound APIs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2010-08 | Initial release by Michael Bleigh (Intridea). |
| 0.19.2 | 2017-04 | mustermann-grape 1.0 router patterns[^2]. |
| 1.0.0 | 2017-07-03 | API stabilization after seven years of 0.x[^2]. |
| 1.3.0 | 2020-01-11 | Virtus replaced with dry-types for coercion[^2]. |
| 2.0.0 | 2023-11-11 | Rack 3 support, Rails 7.1, `Rack::Auth::Digest` removed[^2]. |
| 2.1.0 | 2024-06-15 | Zeitwerk autoloading, `contract` DSL[^2]. |
| 3.0.0 | 2025-11-15 | Ruby 2.7 / ActiveSupport 6.1 dropped; `return` in endpoints deprecated[^2]. |
| 3.3.2 | 2026-07-05 | Current stable[^2]. |
| 4.0.0 | in dev | Ruby 3.3+ required; mustermann-grape inlined; endpoint/router internals reworked[^2]. |

## References

[^1]: Grape README. https://github.com/ruby-grape/grape#readme
[^2]: Grape CHANGELOG. https://github.com/ruby-grape/grape/blob/master/CHANGELOG.md
[^3]: grape-entity — serialization layer. https://github.com/ruby-grape/grape-entity
[^4]: grape-swagger — OpenAPI generation. https://github.com/ruby-grape/grape-swagger
[^5]: Grape issue #1515 — Rack::Cascade 404 ordering. https://github.com/ruby-grape/grape/issues/1515

## Tags

ruby, rack, rest-api, api-framework, dsl, microframework, json-api, api-versioning, openapi, parameter-validation
