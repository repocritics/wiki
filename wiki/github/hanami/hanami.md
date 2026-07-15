# hanami/hanami

> A full-stack Ruby web framework assembled from small single-purpose libraries — the explicit, dependency-injected alternative to Rails.

[GitHub repo](https://github.com/hanami/hanami) ·
[Official website](https://hanamirb.org) ·
[License: MIT](https://github.com/hanami/hanami/blob/main/LICENSE)

## Overview

Hanami is a full-stack Ruby web framework built as a thin layer of glue over a set of independent components: `hanami-router` (Rack routing), `hanami-action` (class-based actions), `hanami-view` (view/template separation), `hanami-db` (persistence), and `hanami-assets` (front-end assets)[^1]. Each can be used on its own; the `hanami` gem is what ties them into an application. It began as **Lotus**, created by Luca Guidi around 2014, and was renamed Hanami in early 2016[^2]. Version 1.0 shipped in April 2017.

The defining fact about modern Hanami is that **2.0 (November 2022) was a ground-up rewrite**, not an iteration[^3]. It replaced the 1.x internals with the dry-rb stack — `dry-system` for the dependency-injection container, plus `dry-configurable`, `dry-monads`, `dry-types`, and `dry-validation` — and adopted ROM (Ruby Object Mapper) over Sequel for the database layer. Where Rails leans on convention, ActiveRecord, and implicit magic, Hanami 2 leans on explicit boundaries, a boot-time container, immutable structs, and a data-mapper persistence model. That is the framework's central tension: it trades Rails' "it just works" velocity and enormous ecosystem for architecture that stays legible as an app grows.

The 2.0 release deliberately shipped **API-first** — it had no view or persistence layer at launch. Those arrived incrementally: views and assets in 2.1 (February 2024), and `Hanami::DB` in 2.2 (November 2024), which is when 2.x finally reached full-stack parity with the 1.x line[^4]. This staged rollout is worth knowing because a lot of production Hanami 2 apps were built API-only on 2.0 before the rest existed.

## Getting Started

```shell
gem install hanami
hanami new bookshelf
cd bookshelf && bundle
bundle exec hanami dev
# Now visit http://localhost:2300
```

A minimal action wires an injected dependency and renders a view. Actions are one class per endpoint:

```ruby
# app/actions/books/index.rb
module Bookshelf
  module Actions
    module Books
      class Index < Bookshelf::Action
        include Deps["repositories.book_repo"]  # dependency injection

        def handle(request, response)
          response[:books] = book_repo.all
        end
      end
    end
  end
end
```

## Architecture / How It Works

At boot, the Hanami app is a **`dry-system` container**. Components (actions, repositories, views, and your own classes) are auto-registered by their file path, and dependencies are pulled in explicitly via the `Deps[...]` mixin rather than referenced as global constants. This is the single biggest departure from Rails: there is no implicit global object graph, and wiring is inspectable.

Key pieces:

- **Slices** — a slice is a sub-application inside the same codebase, with its own container, actions, and boundary. This is Hanami's answer to the "modular monolith": you can carve an app into `admin`, `api`, `main` slices that stay isolated but deploy together, and later extract one if needed.
- **Router / Actions** — `hanami-router` is a Rack-compatible router; actions are plain classes with a `handle(request, response)` method, which makes them trivial to unit-test without a full HTTP stack.
- **View layer** — `hanami-view` (descended from `dry-view`) separates the *view object* (the Ruby logic) from the *template* (ERB/Haml/Slim). Templates get an explicit, scoped context instead of reaching into controller instance variables.
- **Persistence** — `Hanami::DB` is built on ROM over Sequel. It is **data-mapper, not active-record**: relations describe queries, repositories expose them, and rows come back as immutable structs — there is no `Book.save`-style row object that knows how to persist itself.
- **Functional tendencies** — `dry-monads` (`Result`, `Maybe`) and `dry-validation` contracts are idiomatic for business logic, pushing toward explicit error values over exceptions.

The org has since consolidated the Hanami, dry-rb, and rom-rb projects under a shared umbrella (Hanakai), reflecting how tightly the framework now depends on those libraries[^5].

## Production Notes

**The dry-rb stack is a prerequisite, not an implementation detail.** To be productive you have to understand `dry-system` (how components register and resolve), `dry-monads`, and often `dry-validation`. Debugging a boot failure means reading container errors, not Rails-style stack traces. Budget real ramp-up time for a team coming from Rails.

**Data-mapper persistence has a learning curve.** ROM's relations-vs-repositories split, immutable structs, and explicit mapping are more upfront work than ActiveRecord. Developers reflexively looking for `Model.where(...).update` will need to unlearn it. The payoff is that persistence stays decoupled from domain objects.

**Ecosystem size is the real operating constraint.** There are far fewer Hanami-specific gems, tutorials, Stack Overflow answers, and job-market-ready hires than for Rails. Many problems that have a drop-in Rails gem require either a generic Rack gem or hand-rolling. Evaluate this before committing a team.

**No smooth 1.x → 2.x upgrade.** Because 2.0 is a rewrite on a different foundation, moving a Hanami 1 app forward is effectively a re-port, not a `bundle update`. Treat 1.x apps as a separate lineage.

**Full-stack maturity is recent.** Views/assets (2.1) and DB (2.2) only landed in 2024. Some rough edges and thin documentation still show in the newer layers relative to the very stable router/action core.

## When to Use / When Not

**Use when:**
- You're building a long-lived app where architectural clarity and testability matter more than day-one velocity.
- You want a modular monolith with real boundaries (slices) rather than a big ball of Rails.
- You prefer explicit dependency injection and data-mapper persistence over ActiveRecord magic.
- You value functional-style error handling (`dry-monads`) and contract-based validation.

**Avoid when:**
- You need the largest possible ecosystem, hiring pool, and gem availability — Rails wins outright.
- You're prototyping fast and throwaway; the explicitness is overhead you don't need yet.
- Your team has no appetite for learning the dry-rb/ROM stack.
- You want ActiveRecord specifically, or depend on Rails-only gems.

## Alternatives

- rails/rails — batteries-included, ActiveRecord, dominant ecosystem and hiring pool; use it when convention-over-configuration velocity and gem availability outweigh architectural purity.
- jeremyevans/roda — a routing-tree toolkit you compose into your own stack; use it when you want maximum control and minimal dependencies rather than a framework.
- sinatra/sinatra — a micro DSL for small apps and simple APIs; use it when the app is small enough that Hanami's structure would be overhead.
- rom-rb/rom — the data-mapper persistence layer Hanami's DB is built on; use it directly when you want that persistence model in a different framework.
- dry-rb/dry-system — the DI container underneath Hanami; use it when you want the architecture (containers, injection) without adopting a full web framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| Lotus 0.1 | 2014-06 | First release under the original "Lotus" name[^2]. |
| Renamed Hanami | 2016-01 | Project renamed from Lotus to Hanami. |
| 1.0 | 2017-04-06 | First stable full-stack release of the 1.x line. |
| 1.3 | 2018-10 | Final 1.x series before the rewrite. |
| 2.0 | 2022-11-22 | Ground-up rewrite on dry-rb + ROM; API-first, no view/DB yet[^3]. |
| 2.1 | 2024-02 | View and assets layers added (front-end via esbuild)[^4]. |
| 2.2 | 2024-11 | `Hanami::DB` persistence; full-stack parity restored[^4]. |

## References

[^1]: hanami/hanami README — component overview. https://github.com/hanami/hanami
[^2]: Hanami project history (formerly Lotus, renamed 2016). https://hanamirb.org/
[^3]: "Hanami 2.0" release announcement — 2022-11-22. https://hanamirb.org/blog/2022/11/22/announcing-hanami-200/
[^4]: Hanami release blog (2.1 views/assets, 2.2 database). https://hanamirb.org/blog/
[^5]: Hanakai — shared org for Hanami, dry-rb, and rom-rb. https://hanamirb.org/
## Tags

ruby, web-framework, full-stack, dependency-injection, modular-monolith, dry-rb, rom, data-mapper, mvc, rack, api
