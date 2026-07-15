# joke2k/faker

> Python library that generates realistic-looking fake data — names, addresses, text, and hundreds of other field types across many locales.

[GitHub repo](https://github.com/joke2k/faker) ·
[Official docs](https://faker.readthedocs.io) ·
[License: MIT](https://github.com/joke2k/faker/blob/master/LICENSE.txt)

## Overview

Faker is a Python package for producing plausible synthetic data: person names,
street addresses, emails, phone numbers, lorem-ipsum text, dates, credit-card
numbers, and several hundred other "fakes". The typical uses are seeding a
development database, generating fixtures for tests, stress-filling a datastore,
and anonymizing exported production data. It is a de facto standard in the Python
testing ecosystem and is what `factory_boy` delegates to for value generation[^1].

The project began in 2012 and was originally distributed as `fake-factory`; that
name was deprecated by the end of 2016 and the package was renamed to `Faker`[^2].
It is a port of the same idea as PHP Faker (fzaninotto), Perl's `Data::Faker`, and
Ruby's `faker`, and shares their provider-based design[^2].

The defining tension is that Faker optimizes for *looking real*, not for being
statistically real, unique, or secure. Output is drawn from `random.Random`, so it
is not cryptographically safe; values are not unique unless you explicitly ask; and
data distributions are only loosely weighted toward real-world frequencies. Treat
it as a fixture generator, never as a source of security tokens or guaranteed-unique
keys.

## Getting Started

```bash
pip install Faker
```

```python
from faker import Faker

fake = Faker()            # defaults to en_US
Faker.seed(0)             # reproducible within a pinned Faker version

fake.name()               # 'Norma Fisher'
fake.address()            # '644 Ashley Ford Apt. 843\nSouth ...'
fake.email()              # 'donaldgarcia@example.net'

# Localized, and multi-locale
it = Faker('it_IT')
it.name()                 # 'Sig. Avide Guerra'

multi = Faker(['it_IT', 'en_US', 'ja_JP'])   # multi-locale, since v3.0.0
multi.name()
```

There is also a CLI (`faker name`, `faker -l de_DE address`, `faker -r 3 name`) and
a bundled `pytest` plugin exposing a `faker` fixture[^3].

## Architecture / How It Works

The public entry point `Faker()` is a *proxy* over one or more `Generator` objects
(one per locale). Attribute access like `fake.name` is not a normal method: the
proxy forwards `name` to `Generator.format('name')`, which looks the method up in
the registry of installed **providers**[^4].

- **Providers** are classes subclassing `faker.providers.BaseProvider`. Each bundles
  a family of related fakes (`person`, `address`, `internet`, `company`, `lorem`,
  `date_time`, `credit_card`, ...). At construction time Faker eagerly instantiates
  the full default provider set and merges their methods into a flat namespace.
- **Locales** are resolved by module path: `faker/providers/<name>/<locale>/`. If a
  locale-specific provider does not exist, Faker silently falls back to `en_US`. This
  fallback is convenient but means a partially-translated locale can quietly return
  US-shaped data without warning.
- **Multi-locale** instances keep one `Generator` per locale and pick among them per
  call, optionally weighted. Subscript access (`fake['ja_JP'].name()`) pins a locale.
- **Randomness** is shared: by default every generator uses one process-wide
  `random.Random`, reachable as `from faker.generator import random`. `Faker.seed()`
  seeds this shared instance; `fake.seed_instance()` gives one instance its own RNG.

Two convenience layers wrap the generator. `fake.unique.<method>()` remembers values
already produced for that instance and retries until it finds a new one, raising
`UniquenessException` after a bounded number of attempts. `use_weighting` (constructor
arg, default `True`) makes element selection approximate real-world frequency at the
cost of speed; setting it `False` makes selection uniform and faster.

## Production Notes

- **Not cryptographically secure.** Output comes from `random.Random`. Never use it
  for passwords, tokens, salts, or anything an attacker shouldn't predict.
- **Uniqueness is not free.** Bare `fake.email()` / `fake.name()` *will* collide at
  scale (birthday paradox). Use `fake.unique.…`, but know its keyspace is finite:
  requesting more distinct values than a provider can yield raises `UniquenessException`.
  Only hashable arguments/returns work with `.unique`, and `.unique.clear()` resets it.
- **Seed reproducibility is version-scoped.** The datasets ship with the library and
  change between releases, so a given seed only reproduces the same output on the same
  Faker version. If your tests assert on exact generated strings, pin Faker to a patch
  version — otherwise a routine upgrade silently breaks assertions.
- **Construction is comparatively expensive.** Instantiating `Faker()` loads every
  default provider; multi-locale instances multiply that. Build one instance and reuse
  it (e.g. a session-scoped fixture) rather than constructing per row.
- **Thread safety.** The default shared RNG is not designed for concurrent use across
  threads. For parallel workers, give each its own `seed_instance()`-seeded instance.
- **Locale coverage is uneven.** Many providers only fully exist for `en_US`; the
  silent fallback can leave "localized" data looking American. Verify per-field before
  relying on a non-English locale for realism.
- **Volume, not distribution.** Frequencies are only roughly weighted; do not use
  Faker output to train or benchmark anything that depends on realistic statistics.

## When to Use / When Not

**Use when:**
- Seeding dev/test databases or building fixtures that should look human.
- Anonymizing or scrubbing exported data with same-shaped replacements.
- You need broad, localized field coverage out of the box with `factory_boy` or pytest.

**Avoid when:**
- You need security-grade randomness or unpredictability.
- You need guaranteed-unique keys at large scale without managing `.unique` limits.
- You need statistically faithful synthetic data for analytics/ML training.
- Raw generation speed dominates and you don't need Faker's breadth (see mimesis).

## Alternatives

- lk-geimfari/mimesis — faster pure-Python generator with a different API; use it when
  construction/throughput cost matters and you don't need Faker's provider ecosystem.
- FactoryBoy/factory_boy — model/ORM object factories that call Faker under the hood;
  use it when you need populated Django/SQLAlchemy instances, not bare values.
- HypothesisWorks/hypothesis — property-based testing; use it when you want edge-case
  and adversarial inputs rather than realistic-looking data.
- FakerPHP/Faker — the maintained PHP port of the same concept for PHP projects.
- faker-ruby/faker — the Ruby equivalent for Ruby/Rails test suites.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-11-12 | First release as `fake-factory`[^5]. |
| — | 2016 | Renamed `fake-factory` → `Faker`; old package deprecated[^2]. |
| 3.0.0 | 2019 | Multiple-locale support on a single instance[^2]. |
| 4.0.0 | 2020 | Dropped Python 2 support[^2]. |
| 5.0.0 | 2020 | Minimum Python raised to 3.8+[^2]. |

(Version dates before the rename are approximate; the `fake-factory` lineage predates
the current changelog. Pin to the docs changelog for exact release dates[^6].)

## References

[^1]: factory_boy Faker integration. https://factoryboy.readthedocs.io/en/stable/#faker
[^2]: Faker README — compatibility, credits, and localization. https://github.com/joke2k/faker/blob/master/README.rst
[^3]: Faker pytest fixtures. https://faker.readthedocs.io/en/stable/pytest-fixtures.html
[^4]: Faker providers overview. https://faker.readthedocs.io/en/stable/providers.html
[^5]: GitHub repository metadata (created 2012-11-12). https://github.com/joke2k/faker
[^6]: Faker changelog. https://github.com/joke2k/faker/blob/master/CHANGELOG.md

## Tags

python, test-data, fake-data, fixtures, testing, data-generation, anonymization, faker, localization, factory-boy
