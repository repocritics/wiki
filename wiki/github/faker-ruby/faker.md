# faker-ruby/faker

> A Ruby library that generates plausible fake data — names, addresses, emails, Lorem text — for tests, seeds, and demos.

[GitHub repo](https://github.com/faker-ruby/faker) ·
[License: MIT](https://github.com/faker-ruby/faker/blob/main/LICENSE.txt)

## Overview

Faker is one of the oldest and most widely used Ruby gems for fabricating
test data. The repository dates to 2008[^1] and was originally maintained
by Benjamin Curtis under `stympy/faker`; it now lives under the community
`faker-ruby` organization. It is a direct spiritual port of Perl's
`Data::Faker` library[^2], and predates the better-known Python (`joke2k/faker`)
and JavaScript (`faker-js/faker`) libraries that share the name but no code.

Its job is narrow and well-defined: expose a large catalog of generator
classes (`Faker::Name`, `Faker::Internet`, `Faker::Address`, `Faker::Lorem`,
and a few hundred more) whose methods return random, realistic-looking
strings. It is used mostly to populate `db/seeds.rb`, to fill FactoryBot /
fixtures with non-repeating values, and to make demo screenshots look real.

The defining tension is scope versus maintenance. Faker accumulated an
enormous surface of niche generators (Pokemon, Dune quotes, cannabis strains,
dozens of TV shows) contributed over 15 years. As of recent releases the
maintainers explicitly **stopped accepting new generators and new locales**[^3],
freezing the catalog to keep the project maintainable. Faker today is stable
and heavily depended upon, but effectively feature-complete rather than
actively expanding.

## Getting Started

Add it to your Gemfile (typically in the `:test`/`:development` group):

```ruby
gem 'faker'
```

Then `bundle install` and call generators directly:

```ruby
require 'faker'

Faker::Name.name                      #=> "Christophe Bartell"
Faker::Internet.email                 #=> "eliza@mann.test"
Faker::Address.full_address           #=> "5479 William Way, East Sonnyhaven, LA 63637"
Faker::Lorem.paragraph                #=> "Recusandae minima consequatur. Expedita sequi blanditiis."
Faker::Alphanumeric.alpha(number: 10) #=> "zlvubkrwga"

# Guaranteed-unique values within a run:
Faker::Name.unique.name

# Reproducible output via a seeded PRNG:
Faker::Config.random = Random.new(42)
Faker::Lorem.word                     #=> "velit"
```

## Architecture / How It Works

Faker is pure Ruby with two runtime dependencies of note: **I18n** (data
storage) and its own PRNG plumbing. Every generator is a subclass of
`Faker::Base`. The base class provides the string-shaping primitives —
`fetch` (pick a random entry from a data key), `parse` (expand a template),
`numerify`, `letterify`, and `bothify` (fill `#` and `?` placeholders) — and
the concrete generators are mostly thin wrappers that call these against
named data keys.

The actual data lives not in Ruby but in **locale YAML files** under
`lib/locales/`, loaded through the I18n gem. `Faker::Name.name` ultimately
resolves an I18n key like `faker.name.name` against the currently configured
locale. This is why Faker ships 40+ locales and why localization is a
first-class feature: switching `Faker::Config.locale = :es` re-points every
generator at Spanish data without changing call sites.

Two pieces of global state sit in `Faker::Config`:

- **`locale`** — the active I18n locale. When a key is missing in the chosen
  locale, I18n falls back (usually to `:en`), which is why some non-English
  generators silently return English data.
- **`random`** — the PRNG. All randomness routes through this object, so
  assigning a seeded `Random.new(n)` makes the entire library deterministic.
  This is the mechanism behind reproducible test failures.

Uniqueness is a decorator, not a core feature. `Faker::Name.unique` returns a
`UniqueGenerator` proxy that remembers every value it has handed out and
retries the underlying generator until it finds a new one, raising
`RetryLimitExceeded` when the value space is exhausted.

## Production Notes

**"Fake" data can be real.** The README warns explicitly: generated names,
emails, addresses, and phone numbers may coincide with real, valid
information[^4]. Do not use Faker output to hit live endpoints, send test
email, or seed anything user-visible without sanitizing. Prefer the reserved
`.test`/`example.com`-style helpers where they exist.

**Uniqueness is per-process and unbounded in memory.** The `unique` proxy
retains all previously returned values for the life of the process. In long
loops or big seed scripts this grows without limit; call
`Faker::Name.unique.clear` or `Faker::UniqueGenerator.clear` between batches
(and between tests) to release it. Small value spaces (e.g. a two-letter
code) hit `RetryLimitExceeded` quickly.

**Minitest + parallelization can defeat determinism.** Since Faker >= 2.22 a
known interaction causes duplicate values under Minitest[^5]; the documented
fix is to reset `Faker::Config.random = Random.new` in `test_helper.rb` /
`rails_helper.rb`. If you seed the PRNG for reproducibility, be aware that
parallel test runners share or reset that global unpredictably.

**Global state is not free-threaded.** `Faker::Config.locale` and `.random`
are process-global. On threaded servers (Puma, Sidekiq) a locale set in one
request affects others; the project documents a locales-README pattern for
threaded environments rather than making Config thread-local.

**Startup cost and drift.** Requiring Faker eagerly wires up the full I18n
locale set, so load time and resident memory are non-trivial in large suites.
Because the catalog was community-grown, generators and data entries have
also been renamed or removed across majors — tests that assert exact Faker
output (rather than shape) can break on upgrade, so pinning `faker` to a known
version is the norm.

## When to Use / When Not

**Use when:**
- You need realistic seed/fixture/demo data in a Ruby or Rails project.
- You want localized fake data (names/addresses/phones) across many locales.
- You need reproducible randomness in tests via a seedable PRNG.
- You want breadth — a generator almost certainly already exists for your domain.

**Avoid when:**
- You need guaranteed-unique or referentially-consistent data at scale — pair
  it with FactoryBot/sequences, or generate deterministically yourself.
- Startup time and memory are tight and you only need a few generators.
- Your stack isn't Ruby — the same-named Python/JS libraries are separate projects.
- You need data that is provably never a real person/address (Faker can't promise that).

## Alternatives

- joke2k/faker — the Python library of the same name; use it when your test suite is Python, not Ruby.
- faker-js/faker — the JavaScript/TypeScript equivalent; use it in Node or browser stacks.
- ffaker — leaner, faster-loading Ruby faker with fewer generators/locales; use when startup cost matters more than breadth.
- thoughtbot/factory_bot — object-factory library, complementary rather than competing; pair it with Faker to build whole records.
- fakerphp/faker — the maintained PHP port (successor to the archived fzaninotto/Faker); use it in PHP projects.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2008-12-24 | Repository created; early gem ported from Perl's Data::Faker[^1][^2]. |
| 2.0 | 2019 | Major release; dropped older Ruby support, restructured generators. |
| 2.22 | 2022 | Introduced the Minitest duplicate-value interaction documented in the README[^5]. |
| 3.0 | 2022 | Major release; further raised minimum Ruby version. |
| — | recent | Maintainers stop accepting new generators and locales[^3]. |

## References

[^1]: faker-ruby/faker repository metadata — created 2008-12-24. https://github.com/faker-ruby/faker
[^2]: README, "Inspiration" — Faker was inspired by Perl's Data::Faker library. https://metacpan.org/pod/Data::Faker
[^3]: README, "Contributing" — "We are not accepting proposals for new generators and locales." https://github.com/faker-ruby/faker/blob/main/CONTRIBUTING.md
[^4]: README, "Note" — generated data might return valid real information; use with care in tests. https://github.com/faker-ruby/faker
[^5]: README, "Minitest and Faker >= 2.22" — reset `Faker::Config.random` to avoid duplicate values. https://github.com/faker-ruby/faker/issues/2534

## Tags

ruby, testing, test-data, fake-data, fixtures, seed-data, i18n, localization, factory, gem
