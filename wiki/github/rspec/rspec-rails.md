# rspec/rspec-rails

> RSpec for Ruby on Rails — the BDD spec framework that half the Rails world uses instead of the built-in Minitest stack.

[GitHub repo](https://github.com/rspec/rspec-rails) ·
[Official website](https://rspec.info) ·
[License: MIT](https://github.com/rspec/rspec-rails/blob/main/LICENSE.md)

## Overview

rspec-rails is the Rails adapter for RSpec, the behavior-driven testing framework for Ruby. It replaces Rails' default Minitest-based test stack with RSpec's `describe`/`it`/`expect` DSL while reusing Rails' own test infrastructure underneath — every rspec-rails spec type wraps a corresponding Rails `TestCase` class rather than reimplementing it[^1]. The repo dates to 2009 on GitHub (the 1.x line lived earlier under dchelimsky/rspec-rails), making it one of the oldest continuously maintained gems in the Rails ecosystem.

The 5.3k stars understate its footprint: testing libraries under-index on stars, and rspec-rails predates star culture. In practice RSpec vs Minitest is the longest-running tooling schism in Rails — DHH and Rails core favor Minitest's plain-Ruby minimalism; a large fraction of production Rails shops (and nearly all consultancies of the thoughtbot lineage) standardized on RSpec's readable spec output and richer matcher vocabulary. The project is actively maintained (pushed June 2026) with a deliberate cadence: since the versioning-strategy RFC, major versions track supported Rails versions rather than RSpec core versions[^2] — 8.x for Rails 7.2/8.0, 7.x for Rails 7.x, 6.x for Rails 6.1–7.1.

The defining tension: rspec-rails buys expressive, self-documenting specs at the cost of a second DSL layered on Rails, slower suite boot than bare Minitest, and a version matrix you must keep aligned with your Rails upgrade path.

## Getting Started

```ruby
# Gemfile — both groups, so generators work without RAILS_ENV=test
group :development, :test do
  gem "rspec-rails", "~> 8.0"
end
```

```sh
bundle install
rails generate rspec:install   # creates .rspec, spec/spec_helper.rb, spec/rails_helper.rb
```

```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  context "before publication" do
    it "cannot have comments" do
      expect { Post.create.comments.create! }.to raise_error(ActiveRecord::RecordInvalid)
    end
  end
end
```

```sh
bundle exec rspec                       # whole suite
bundle exec rspec spec/models           # one directory
bundle exec rspec spec/models/post_spec.rb:5   # one example by line
```

## Architecture / How It Works

rspec-rails is a thin integration layer, not a test framework — the framework is rspec-core + rspec-expectations + rspec-mocks (now developed in the rspec/rspec monorepo; rspec-rails remains a separate repo).

- **Spec types via metadata.** `type: :model`, `:request`, `:system`, etc. (ten types total) mix in a module that adapts the matching Rails test class: request specs wrap `ActionDispatch::IntegrationTest`, system specs wrap `ActionDispatch::SystemTestCase`, controller specs wrap `ActionController::TestCase`[^1]. Rails' own helper methods (`get`, `post`, `travel_to`, fixtures) are therefore all available inside specs. Minitest setup/teardown hooks are bridged through an internal adapter. Since RSpec 3, inferring type from file location (`spec/models/` → `:model`) is opt-in via `infer_spec_type_from_file_location!`.
- **Two helper files.** `spec_helper.rb` configures bare RSpec; `rails_helper.rb` boots the Rails environment and requires `spec_helper`. Unit specs that don't touch Rails can require only `spec_helper` and skip the multi-second boot — a split introduced in rspec-rails 3.0 precisely for this[^3].
- **Generator integration.** A Railtie hooks RSpec into `rails generate`: scaffolding a model emits `spec/models/user_spec.rb` instead of a Minitest file, and `rails generate rspec:*` generators exist for every spec type.
- **Rails-specific matchers.** `redirect_to`, `render_template`, `route_to`, `have_http_status`, `have_enqueued_job` / `have_been_enqueued` (ActiveJob), `have_enqueued_mail`, `have_broadcasted_to` (ActionCable) — mostly thin delegations to Rails assertions (`assert_redirected_to`, `assert_recognizes`), re-expressed as composable matchers.
- **Transactional examples.** Each example runs inside a rolled-back DB transaction (`use_transactional_fixtures`). For system specs, Rails 5.1+ shares the DB connection between the app server thread and the test thread, so transactions work with a real browser without DatabaseCleaner.

## Production Notes

**The version matrix is your upgrade tax.** Major rspec-rails versions map to Rails support windows[^2]. A Rails upgrade routinely forces an rspec-rails major bump, which can pull new rspec-core majors and matcher deprecations into the same diff as your framework upgrade. Pin deliberately and upgrade rspec-rails first, on the old Rails version, when possible.

**Controller specs are legacy.** Since RSpec 3.5 / Rails 5, both teams discourage controller specs in favor of request specs[^4]. `assigns` and `assert_template` were extracted from Rails into the rails-controller-testing gem — if a legacy suite uses them, you carry an extra dependency. New code should use request specs; they exercise routing and middleware and are what maintainers actually test against.

**No built-in parallelism.** Rails' Minitest stack has had parallel test execution since Rails 6; RSpec does not, and rspec-rails inherits that gap. Large suites reach for grosser/parallel_tests or turbo_tests, both of which need per-process database setup. This is the most concrete argument for Minitest on very large monoliths.

**Boot time dominates small runs.** `bundle exec rspec spec/models/foo_spec.rb` pays full Rails boot. Mitigations: the `spec_helper`-only pattern for pure-Ruby units, Spring (with known staleness footguns), or bootsnap. Suite speed problems are almost never RSpec overhead; they are Rails boot plus per-example DB work.

**System/feature spec configuration.** System specs (Rails 5.1+) come pre-wired with Capybara and driver management; feature specs are the older equivalent that require you to configure JS drivers and DB state sharing yourself. Use system specs unless stuck on old Rails[^5]. Capybara's DSL is only auto-available in those two types — using it elsewhere needs explicit configuration.

**Monkey-patching mode.** Legacy suites may still use the global `describe` and `should` syntax. `config.disable_monkey_patching!` enforces the modern zero-monkey-patch style (`RSpec.describe`, `expect`); mixed suites are a common archaeological layer in older apps.

## When to Use / When Not

**Use when:**
- Your team reads tests as behavior documentation — RSpec's `--format documentation` output and composable matchers are the payoff.
- You rely on the ecosystem built on top of it: shoulda-matchers, FactoryBot conventions, VCR/WebMock integration recipes are RSpec-first.
- You are hiring from the consultancy/product-shop talent pool, where RSpec is the default dialect.

**Avoid when:**
- You want the zero-decision Rails default: Minitest ships with Rails, boots faster, and has built-in parallel execution.
- Your suite is a huge monolith where parallelism is existential — Rails' native parallel Minitest is simpler than bolting parallel_tests onto RSpec.
- The team dislikes DSL-heavy magic; plain-Ruby Minitest tests are easier to debug with ordinary tooling.

## Alternatives

- seattlerb/minitest — use instead when you want Rails' built-in default, faster boot, plain Ruby assertions, and native parallel execution.
- cucumber/cucumber-ruby — use instead when non-developers must read/write acceptance criteria in Gherkin; heavier and largely superseded by system specs for dev-only teams.
- thoughtbot/shoulda-matchers — not a replacement but the standard companion: one-line matchers for validations/associations on top of rspec-rails.
- teamcapybara/capybara — the browser-interaction layer both rspec-rails system specs and Cucumber drive; you use it with rspec-rails, not instead of it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2007-05 | RSpec 1.0 era; Rails integration maintained in dchelimsky/rspec-rails. |
| 2.0 | 2010-10 | RSpec 2 modular rewrite (core/expectations/mocks); Rails 3 support. |
| 3.0 | 2014-06 | `rails_helper`/`spec_helper` split[^3]; zero-monkey-patching mode; spec-type inference made opt-in. |
| 3.5 | 2016-07 | Rails 5; request specs officially preferred over controller specs[^4]. |
| 3.7 | 2017-10 | System specs wrapping `ActionDispatch::SystemTestCase`[^5]. |
| 4.0 | 2020-03 | Versioning decoupled from RSpec core; Rails 5–6 window[^6]. |
| 6.0 | 2022-10 | Rails 6.1–7.0 window. |
| 7.0 | 2024-09 | Rails 7.x window under the Rails-tracking versioning strategy[^2]. |
| 8.0 | 2025 | Rails 7.2 / 8.0 window (current stable line, `8-0-maintenance` branch). |

## References

[^1]: rspec-rails README, "What tests should I write?" — spec type to Rails `TestCase` mapping. https://github.com/rspec/rspec-rails#what-tests-should-i-write
[^2]: rspec-rails versioning strategy RFC. https://github.com/rspec/rspec-rails/blob/main/rfcs/versioning-strategy.md
[^3]: RSpec, "RSpec 3 has been released" — 2014-06. https://rspec.info/blog/2014/06/rspec-3-has-been-released/
[^4]: RSpec, "RSpec 3.5 has been released" — Rails 5 support, controller-spec guidance — 2016-07. https://rspec.info/blog/2016/07/rspec-3-5-has-been-released/
[^5]: RSpec, "RSpec 3.7 has been released" — system spec integration — 2017-10. https://rspec.info/blog/2017/10/rspec-3-7-has-been-released/
[^6]: rspec-rails Changelog, 4.0.0. https://github.com/rspec/rspec-rails/blob/main/Changelog.md
[^7]: rails/rails-controller-testing — extracted `assigns`/`assert_template`. https://github.com/rails/rails-controller-testing

## Tags

ruby, rails, testing, bdd, tdd, rspec, test-framework, dsl, developer-tools
