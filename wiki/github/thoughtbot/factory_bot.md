# thoughtbot/factory_bot

> A Ruby library for building objects as test data — the de facto factories replacement for Rails fixtures.

[GitHub repo](https://github.com/thoughtbot/factory_bot) ·
[Official website](https://thoughtbot.com) ·
[License: MIT](https://github.com/thoughtbot/factory_bot/blob/main/LICENSE)

## Overview

factory_bot is a fixtures replacement with a block-based definition syntax and
several build strategies: persisted instances, unsaved instances, attribute
hashes, and stubbed objects. It lets a test say "give me a valid user" without
restating every required column, and lets multiple factories describe the same
class (`user`, `admin_user`) through inheritance and traits. It is ORM-agnostic
at its core — it works with ActiveRecord, Sequel, or plain Ruby objects — with
the Rails-specific glue living in the companion gem factory_bot_rails[^1].

The library was originally written by Joe Ferris at thoughtbot and released in
2008 as **factory_girl**. It was renamed to factory_bot in 2017; the old gem
name persists as a deprecated shim, and much blog content, Stack Overflow
answers, and third-party integrations still reference it[^2]. Assume
factory_girl and factory_bot are the same project at different points in time.

The defining tension is convenience versus test speed. Factories make it trivial
to conjure a complete, valid object graph, which is exactly why suites built on
them tend to get slow: a single `create(:user)` can cascade through associations
and callbacks into dozens of database inserts nobody asked for. The library
gives you the tools to avoid this (`build`, `build_stubbed`, transient
attributes) but does not stop you from writing the slow version.

## Getting Started

```ruby
# Gemfile — plain Ruby project
bundle add factory_bot
# Rails project (adds generators + auto-loads spec/factories)
bundle add factory_bot_rails
```

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { "Jane Doe" }
    sequence(:email) { |n| "user#{n}@example.com" }
    admin { false }

    trait :admin do
      admin { true }
    end

    factory :admin_user do
      admin { true }
    end
  end
end
```

```ruby
build(:user)              # unsaved instance, no DB hit
create(:user, :admin)     # persisted, admin trait applied
attributes_for(:user)     # plain attribute hash
build_stubbed(:user)      # fake persisted object, still no DB
```

## Architecture / How It Works

`FactoryBot.define` evaluates its block against a DSL that registers factory
definitions in a single global registry (`FactoryBot::Internal`). Definitions
are lazy: attribute blocks are not evaluated until you call a build strategy, so
a `sequence` or an association runs per-instance, not at load time. This is why
factory_bot 5.0 removed the old static syntax (`name "Jane"`) and made the block
form (`name { "Jane" }`) mandatory — static values were evaluated once and
shared, a persistent source of surprising test bugs[^3].

The four build strategies differ only in what they do after constructing the
object. `build` news up the instance and runs `after(:build)` callbacks.
`create` additionally persists it and runs `before`/`after(:create)` callbacks.
`attributes_for` returns a hash and skips object construction. `build_stubbed`
constructs the object, assigns a fake ID, and stubs ActiveRecord persistence
methods so that touching the database raises — it is the fastest strategy and
the one most teams underuse.

Composition happens through **traits** (named bundles of attribute overrides
that can be mixed at call time), **associations** (which recursively invoke
another factory), **transient attributes** (inputs that shape the build but are
not set on the object), **sequences** (monotonic counters for uniqueness), and
**callbacks**. Associations are the coupling story: a factory that declares
`association :account` will, under `create`, create an account, which may create
its own owner, and so on. The graph is implicit and easy to grow without
noticing.

## Production Notes

**The factory cascade is the number-one performance problem.** Because `create`
follows associations, an innocuous `create(:comment)` can insert a comment, a
post, a user, and an account. Audit factories so associations use `build` where
possible, prefer `build_stubbed` in unit tests, and reach for `create` only when
persistence is under test. Suites that ignore this routinely spend the majority
of their wall-clock time in factory-driven inserts.

**Lint your factories in CI.** `FactoryBot.lint` instantiates every factory and
every trait and raises on the ones that no longer produce valid objects. Without
it, a factory rots silently as validations change and you only discover it when
an unrelated test fails. The linter is slow (it exercises the whole registry), so
most teams run it as a separate CI step rather than in the main suite.

**Global mutable state.** Sequences persist across examples within a process and
are not reset between tests unless you do so explicitly — relying on a specific
sequence value in an assertion is fragile. All definitions share one registry, so
a duplicate factory name or a typo'd trait fails at load time for the whole
suite. factory_bot_rails auto-loads `spec/factories/*.rb`; bare factory_bot
requires you to load them yourself.

**Upgrade friction.** The 4.x→5.0 jump (removing static attributes) is the most
disruptive migration in the project's history and still trips up suites moving
off very old factory_girl definitions. Newer majors have mostly tracked Ruby and
Rails version support rather than changing the DSL.

## When to Use / When Not

**Use when:**
- You have non-trivial validations and want tests to build valid objects without
  restating every field.
- You need several variants of one model (traits/inheritance) and readable,
  intention-revealing setup.
- You want stubbed or unsaved objects for fast unit tests via `build_stubbed`.

**Avoid when:**
- Your data is genuinely static and shared across the suite — Rails fixtures load
  once and are dramatically faster.
- You are tempted to `create` deep graphs everywhere; that path leads to a slow,
  DB-bound suite regardless of the tool.
- You want randomized field values specifically — pair factory_bot with a data
  generator rather than expecting it to invent realistic strings.

## Alternatives

- rails/rails — built-in fixtures; use when data is static and suite speed
  matters more than flexibility.
- paulelliott/fabrication — competing Ruby factory library with a different DSL
  and lazy-loading model; use if you prefer its syntax.
- faker-ruby/faker — complementary, not a replacement; generates realistic field
  values to feed into factory blocks.
- thoughtbot/factory_bot_rails — the Rails integration wrapper; use this (not
  bare factory_bot) inside a Rails app.
- FactoryBoy/factory_boy — the Python port of the same idea, for polyglot teams
  wanting a consistent mental model.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 (factory_girl) | 2008 | Initial release by Joe Ferris at thoughtbot[^2]. |
| 4.0 | 2012 | Mature factory_girl era; traits, sequences, callbacks stable. |
| rename | 2017-10 | factory_girl renamed to factory_bot; old gem kept as shim[^2]. |
| 5.0 | 2019 | Removed static attribute syntax; block form now required[^3]. |
| 6.0 | 2020 | Dropped older Ruby support; maintenance-focused major. |

## References

[^1]: factory_bot README and factory_bot_rails. https://github.com/thoughtbot/factory_bot_rails
[^2]: Project name history (factory_girl → factory_bot). https://github.com/thoughtbot/factory_bot/blob/main/NAME.md
[^3]: factory_bot CHANGELOG / GETTING_STARTED — build strategies and syntax changes. https://github.com/thoughtbot/factory_bot/blob/main/GETTING_STARTED.md

## Tags

ruby, testing, test-data, factories, fixtures, rails, rspec, thoughtbot, mit-license, test-tooling
