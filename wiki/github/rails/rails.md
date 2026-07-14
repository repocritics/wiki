# rails/rails

> The original convention-over-configuration full-stack web framework — a database-backed MVC monolith for Ruby, opinionated by design.

[GitHub repo](https://github.com/rails/rails) ·
[Official website](https://rubyonrails.org) ·
[License: MIT](https://github.com/rails/rails/blob/main/MIT-LICENSE)

## Overview

Ruby on Rails is a full-stack web framework extracted by David Heinemeier Hansson from Basecamp (then 37signals) and released in 2004; version 1.0 shipped in December 2005[^1]. It popularized "Convention over Configuration" and "Don't Repeat Yourself" — the idea that a framework should impose sensible defaults so that following the convention means writing almost no configuration. A generated Rails app wires an ORM, router, view layer, mailer, job queue, WebSocket layer, and asset pipeline together with no glue code from the developer.

Rails is deliberately a monolith. DHH's "majestic monolith" and "omakase" (chef's-choice) framing is a rejection of the assemble-your-own-stack approach: the maintainers pick the components and integrate them, and you are expected to accept the menu. This is the framework's defining tension. Teams that stay on the golden path ship extremely fast; teams that want to swap the ORM, decompose into services early, or fight the conventions spend their time working against the grain. Rails at 20 years old is still the reference implementation of the batteries-included web framework, and the direct ancestor of Laravel, Django's admin ergonomics, and a generation of "Rails for language X" clones.

The engine is **Active Record**[^2], an implementation of Martin Fowler's pattern of the same name: model classes map to database tables, instances map to rows, and business logic lives on the model. This is productive and also the source of most large-app pain — fat models, callback spaghetti, and N+1 queries are Active Record's characteristic failure modes, not accidents of bad code.

## Getting Started

```bash
gem install rails
rails new myapp
cd myapp
bin/rails server   # http://localhost:3000
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  validates :title, presence: true
end

# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.includes(:author).order(created_at: :desc)
  end
end
```

```bash
bin/rails generate model Post title:string body:text author:references
bin/rails db:migrate
```

## Architecture / How It Works

Rails is a collection of gems assembled under one namespace. Each can be used independently[^2]:

- **Active Record** — the ORM. Models subclass `ApplicationRecord`; schema is inferred from the database, associations and validations are declared, and migrations version the schema over time.
- **Action Pack** — routing (Action Dispatch) plus controllers (Action Controller). The router maps URLs to controller actions; controllers load models and render views.
- **Action View** — templates, by default ERB (embedded Ruby). Also drives mailer and Action Text bodies.
- **Active Support** — the utility layer that monkey-patches Ruby core (`2.days.ago`, `blank?`, `Hash#deep_merge`). Ubiquitous and load-bearing across the ecosystem.
- **Active Job / Action Mailer / Action Mailbox / Action Cable / Active Storage / Action Text** — background jobs, outbound/inbound email, WebSockets, file attachments, and rich text, respectively.

**Autoloading** moved to **Zeitwerk** in Rails 6[^3], replacing the older "classic" autoloader. Zeitwerk enforces a strict file-path-to-constant-name mapping; mismatches raise at boot. This eliminated a class of load-order bugs but is a frequent migration snag.

**The front-end story is now Hotwire** — Turbo (HTML-over-the-wire, partial page updates without a client framework) plus Stimulus (a modest JS sprinkles library) — made default in Rails 7[^4]. Import maps let you ship ES modules without a Node bundler at all; `jsbundling-rails` / `cssbundling-rails` remain available for teams that want esbuild, Bun, or Tailwind's toolchain.

**Rails 8 pushed "no PaaS required"**[^5]: Solid Queue, Solid Cache, and Solid Cable replace Redis/Sidekiq-style dependencies with database-backed adapters (SQLite or your primary DB); Kamal 2 handles container deployment to bare servers; Propshaft replaced Sprockets as the default asset pipeline; and a built-in authentication generator ships instead of pulling in Devise. The bet is that a single Rails machine with SQLite can carry a real production app.

The whole stack is co-designed. That coherence is the strength (nothing to wire up) and the coupling risk (deviating from a default means reintegrating a piece the framework assumed it owned).

## Production Notes

**N+1 queries are the default failure mode.** Active Record makes it trivial to iterate a collection and lazily load an association per row. Use `includes` / `preload` / `eager_load`, and add the `bullet` gem in development to catch them. `strict_loading` can make lazy loads raise instead of silently querying.

**Boot time and memory.** A large Rails monolith boots slowly (eager-loading every class in production) and each Puma worker holds a full copy of the app in memory. `jemalloc` noticeably reduces RSS bloat; `pitchfork` (Shopify's forking server with reforking) is the scaling answer at the largest tier. Expect 250–500 MB per worker for a mid-sized app.

**Concurrency and the GVL.** MRI Ruby has a Global VM Lock, so a single process does not run Ruby code on multiple cores in parallel. Rails scales with multiple Puma workers (processes) × threads; threads help mostly for I/O-bound work. Thread-safety of your own code (shared mutable state, class-level caches) is your responsibility — the framework is thread-safe, ad-hoc globals are not.

**Migration pains, in rough order of severity:**
- Ruby version coupling — each Rails major requires a recent Ruby (Rails 8 needs Ruby 3.2+). Upgrades are two-dimensional.
- Classic → Zeitwerk autoloader (Rails 6) forced many apps to rename files to match constants[^3].
- Sprockets → Propshaft (Rails 8 default) changes how digested assets and fingerprinting work; Sprockets-era helpers and directives do not carry over unchanged.
- `default_scope`, model callbacks, and `after_save` side effects are the classic "worked in the small, exploded in the large" traps. Service objects and explicit query scopes are the common escape hatch.

**Testing.** Minitest is the built-in default; RSpec is the dominant community alternative. Fixtures load fast but drift from validations; `factory_bot` is the usual replacement. System tests drive a real browser via Capybara + Selenium and are slow — keep them thin.

**Upgrade cadence.** Rails follows semantic versioning with deprecation warnings one minor release before removal, and ships `bin/rails app:update` to reconcile config. Only the latest two minor series receive bug and security fixes, so staying within ~18 months of current is a security requirement, not just hygiene.

## When to Use / When Not

**Use when:**
- You want a database-backed CRUD-heavy app shipped quickly by a small team.
- You value one integrated, maintained stack over assembling your own.
- Server-rendered HTML with light interactivity (Hotwire) fits the product.
- You want to run on your own boxes without a managed platform (Rails 8 + Kamal + SQLite/Postgres).

**Avoid when:**
- You need CPU-bound parallelism or hard real-time latency — the GVL and per-request object churn work against you.
- Your front end is a heavy SPA and the back end is a thin API; a leaner API framework plus a JS stack has less overhead.
- The team wants to swap Active Record, decompose into services from day one, or otherwise fight the conventions — you lose most of Rails' value.
- You need a compiled, statically typed guarantee across the codebase (Sorbet/RBS help but are bolt-ons).

## Alternatives

- django/django — the Python equivalent; batteries-included, stronger built-in admin, less "magic" than Active Record. Use when your team is Python-first.
- laravel/laravel — the PHP framework most directly inspired by Rails; use it in a PHP shop wanting the same ergonomics.
- phoenixframework/phoenix — Elixir/BEAM; use when you need real concurrency and low-latency WebSockets (LiveView) that the GVL denies Rails.
- sinatra/sinatra — minimal Ruby web toolkit; use for small APIs or services where a full Rails stack is overkill.
- hanami/hanami — Ruby alternative with explicit boundaries and less monkey-patching; use when you want Ruby but dislike Active Record's global conventions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2005-12 | First stable release; convention-over-configuration[^1]. |
| 2.0 | 2007-12 | RESTful routing, multibyte support. |
| 3.0 | 2010-08 | Merb merge; new router, Bundler, Active Relation. |
| 4.0 | 2013-06 | Strong parameters, Turbolinks, Russian-doll caching. |
| 5.0 | 2016-06 | Action Cable (WebSockets), API-only mode. |
| 6.0 | 2019-08 | Zeitwerk autoloader, multi-DB, Action Text/Mailbox[^3]. |
| 7.0 | 2021-12 | Hotwire (Turbo + Stimulus) default, import maps, encrypted attributes[^4]. |
| 7.1 | 2023-10 | Dockerfiles by default, composite primary keys, async queries. |
| 8.0 | 2024-11 | Solid Queue/Cache/Cable, Propshaft, Kamal 2, built-in auth[^5]. |

## References

[^1]: Ruby on Rails release history and origin (extracted from Basecamp by DHH, 2004; 1.0 in December 2005). https://rubyonrails.org/2005/12/13/rails-1-0-party-like-its-one-oh-oh
[^2]: Rails README and component guide — Active Record, Action Pack, Action View, Active Support, and the Action*/Active* libraries. https://github.com/rails/rails/blob/main/README.md
[^3]: Rails 6.0 release notes — Zeitwerk, multiple databases, Action Mailbox, Action Text. https://guides.rubyonrails.org/6_0_release_notes.html
[^4]: Rails 7.0 release notes — Hotwire default, import maps, at-rest encryption. https://guides.rubyonrails.org/7_0_release_notes.html
[^5]: Rails 8.0 release notes — Solid Queue/Cache/Cable, Propshaft, Kamal 2, authentication generator. https://guides.rubyonrails.org/8_0_release_notes.html

## Tags

ruby, web-framework, mvc, orm, active-record, full-stack, monolith, convention-over-configuration, backend, hotwire
