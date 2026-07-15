# flyerhzm/bullet

> A development-time profiler that watches ActiveRecord/Mongoid queries and warns you about N+1 queries, unused eager loading, and missing counter caches.

[GitHub repo](https://github.com/flyerhzm/bullet) ·
[RubyGems](https://rubygems.org/gems/bullet) ·
[License: MIT](https://github.com/flyerhzm/bullet/blob/main/MIT-LICENSE)

## Overview

Bullet is a Ruby gem, first released in 2009 by Richard Huang (flyerhzm), that instruments a Rails or Rack application during development and flags three query-level anti-patterns: N+1 queries (add eager loading), unnecessary eager loading (remove `includes`), and associations that would benefit from a `counter_cache`[^1]. It is one of the oldest and most widely-installed pieces of Rails development tooling, and for most Rails teams it is the default answer to "how do I catch N+1 queries."

The defining design choice is that Bullet is a *runtime* detector, not a static analyzer. It hooks into ActiveRecord's association-loading machinery and records, per HTTP request, which records were loaded and which associations were subsequently accessed. This makes it accurate — it reports the queries your code actually ran on the path you exercised — but also means it only sees code paths you actually hit. It catches nothing you did not browse to or write a test for. It is a magnifying glass held over one request at a time, not a linter.

The intended posture is loud-in-development, silent-in-production. The README is explicit that Bullet belongs in development or a custom profiling environment, not in production where end users would see the alerts[^1]. It supports ActiveRecord >= 4.0 and Mongoid >= 4.0; older Rails require pinned older Bullet versions[^2].

## Getting Started

```ruby
# Gemfile
gem 'bullet', group: 'development'
```

```bash
bundle install
bundle exec rails g bullet:install   # writes default config into development.rb
```

```ruby
# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert  = true          # JS alert in the browser
  Bullet.bullet_logger = true   # log/bullet.log
  Bullet.rails_logger  = true   # inline in the Rails log
  Bullet.add_footer    = true   # detail panel in the page corner
end
```

Bullet notifies nothing until `Bullet.enable = true` is set. It ships integrations for a long list of sinks (Sentry, Honeybadger, Bugsnag, Rollbar, Airbrake, AppSignal, Slack, OpenTelemetry, browser console, XMPP), all off by default and toggled individually[^1].

## Architecture / How It Works

Bullet's core is a set of monkey-patches applied to ActiveRecord (and Mongoid) internals via a small association layer. When an association is defined as loadable and then eager-loaded via `includes`/`preload`/`eager_load`, Bullet records it; when a lazily-loaded association is walked in a loop, it records that too. At the end of a request it diffs "what was eager loaded" against "what was actually accessed" to produce the three detector outputs. The detectors — `n_plus_one_query`, `unused_eager_loading`, `counter_cache` — can each be disabled independently[^1].

The request lifecycle is bracketed by `Bullet.start_request` / `Bullet.end_request`. In a Rails app this is handled automatically by `Bullet::Rack`, a middleware inserted into the stack. That is the key coupling detail with practical consequences: **anything that does not pass through the Rack middleware is not instrumented automatically.** Controller and integration tests work because they go through Rack; model unit tests, background jobs, and console sessions do not, and must be wrapped manually with `Bullet.start_request`/`end_request` or `Bullet.profile { ... }`[^1].

For browser-facing feedback, `Bullet::Rack` injects a `<script>` and optional footer HTML into text/html responses, and sets HTTP headers that a small JS payload reads to update the console/footer. This HTML/header injection is the source of most of Bullet's operational friction (see Production Notes). Notification delivery is delegated to a sibling gem, `uniform_notifier`, which owns the actual adapters for Slack, Sentry, XMPP, and the rest[^3].

Thread-safety was historically a weak point. Current guidance is that `Bullet.enable` is a global, non-thread-safe flag that must not be toggled at runtime; for per-request or per-action suppression you use `Bullet.skip { ... }`, or `Bullet.pause`/`Bullet.resume`, which are thread-local and safe under threaded servers like Puma[^1].

## Production Notes

Bullet is a development tool, but it still has real operational footguns:

- **HTML injection breaks things.** Because Bullet rewrites HTML responses to add its footer/console script, it can corrupt responses, interfere with Content-Security-Policy, or surprise SPA/JSON front-ends. `Bullet::Rack` must be loaded *before* any CSP-generating middleware[^1]. For API-only or SPA setups, `Bullet.skip_html_injection = true` and `Bullet.skip_http_headers = true` exist — but enabling them disables the browser console and footer, leaving only the log file.
- **False positives are common and expected.** Associations loaded by gems, admin panels, or intentionally lazy paths trigger warnings you do not want to fix. The `Bullet.add_safelist` API (keyed by type + class_name + association) is the escape hatch, and a real Bullet adoption on a mature codebase usually means curating a safelist, not reaching zero warnings[^1].
- **`Bullet.raise = true` in tests is powerful and blunt.** Raising on detection is the idiom for failing specs on N+1 regressions, but it turns every unsafelisted detection — including legitimate ones in third-party code paths — into a hard test failure. Teams typically pair it with a growing safelist.
- **It only sees exercised paths.** Coverage equals whatever your test suite or manual browsing touches. Bullet gives no guarantee about routes you did not hit, so it complements but does not replace query review.
- **Browser cache masks results.** The README's own "Important" note: if Bullet appears not to work, disable the browser cache — cached HTML skips the injected script[^1].
- **Version pinning by Rails age.** ActiveRecord 2.x needs bullet <= 4.5.0; 3.x needs bullet < 5.5.0. Upgrading Rails and Bullet together is the norm[^2].

## When to Use / When Not

**Use when:**
- You run Rails/ActiveRecord (or Mongoid) and want to catch N+1 and over-eager-loading during development.
- You want CI to fail on N+1 regressions (`Bullet.raise` in the test env, wrapped per test).
- You want zero-config, per-request feedback while browsing a dev server.

**Avoid / look elsewhere when:**
- You are not on ActiveRecord or Mongoid — Bullet is tied to those ORMs and their internals.
- You need static, whole-codebase analysis independent of exercised code paths — Bullet is runtime-only.
- You need production query monitoring — that is an APM's job, not Bullet's; running it in production is explicitly discouraged.
- Your app is API-only and you will not use the log/notifier sinks; the value drops once HTML injection is off.

## Alternatives

- rails/rails — `strict_loading` (Rails 6.1+) raises on lazy association loads at the model/association level, catching N+1 without a separate gem, though with less nuance than Bullet's three detectors.
- prosopite (charkost/prosopite) — runtime N+1 detector that catches cases Bullet misses (e.g. queries with differing bind values) with fewer false negatives; use instead when Bullet's accuracy is not enough.
- jonleighton/goldiloader — automatically eager-loads associations to eliminate N+1 rather than warn about it; use when you want the fix applied instead of a warning.
- prathamesh-sonpatki/query_diet or rack-mini-profiler (MiniProfiler/rack-mini-profiler) — broader request-level SQL/timing profiling; use when you want general query visibility rather than N+1-specific detection.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2009-08 | First release by Richard Huang; featured in Rails community highlights[^4]. |
| 4.5.0 | — | Last line supporting ActiveRecord 2.x[^2]. |
| 5.x | — | Requires ActiveRecord >= 4.0 / Mongoid >= 4.0; 5.5.0 drops ActiveRecord 3.x[^2]. |
| 7.x | 2020s | Adds OpenTelemetry, thread-safe `skip`/`pause`/`resume`, safelist API, footer positioning. |

(Bullet does not maintain prominent dated release notes in its README; exact per-version dates are best read from the RubyGems version history and CHANGELOG[^5].)

## References

[^1]: Bullet README — configuration, detectors, safelist, skip/pause, Rack, testing. https://github.com/flyerhzm/bullet/blob/main/README.md
[^2]: Bullet README, ActiveRecord/Mongoid version support notes. https://github.com/flyerhzm/bullet#readme
[^3]: uniform_notifier — notification adapter gem used by Bullet. https://github.com/flyerhzm/uniform_notifier
[^4]: Ruby on Rails community highlights, 2009-10-22. https://rubyonrails.org/2009/10/22/community-highlights
[^5]: Bullet on RubyGems (version history / CHANGELOG). https://rubygems.org/gems/bullet

## Tags

ruby, rails, activerecord, n-plus-one, database, orm, performance, profiling, development-tool, mongoid, rack-middleware
