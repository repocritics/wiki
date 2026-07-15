# FakerPHP/Faker

> Community-maintained PHP library that generates fake data — names, addresses, text, records — for seeding databases, fixtures, and anonymization.

[GitHub repo](https://github.com/FakerPHP/Faker) ·
[Official website](https://fakerphp.github.io) ·
[License: MIT](https://github.com/FakerPHP/Faker/blob/2.0/LICENSE)

## Overview

Faker generates realistic-looking synthetic data: person names, emails, addresses,
lorem-ipsum text, dates, numbers, and dozens of other domain types. Its typical
jobs are seeding development databases, producing test fixtures, stress-filling a
persistence layer, and anonymizing production data. It is the de facto fake-data
library in the PHP ecosystem — Laravel's model factories, for example, expose a
Faker instance directly. It is heavily inspired by Perl's `Data::Faker` and Ruby's
Faker, and the API mirrors those closely.

This repository is the maintained continuation of the original `fzaninotto/Faker`,
which its author archived in late 2020 after years as the standard[^1]. FakerPHP
picked up the abandoned codebase, kept the package name namespace-compatible
(`Faker\Factory`, `fakerphp/faker` on Packagist), and has carried maintenance,
PHP-version support, and bugfixes since[^2]. If you see `fzaninotto/faker` in a
`composer.json`, it is the dead upstream; `fakerphp/faker` is the live one.

The defining tension is that Faker is a convenience tool, not a correctness tool.
Its output is random and its data type — not its exact value — is the only thing
the project promises to keep stable across versions[^3]. Treating Faker output as
a fixed, snapshot-testable value, or leaning on it for anything security-adjacent,
is where teams get burned.

## Getting Started

Faker requires PHP >= 7.4[^2].

```shell
composer require fakerphp/faker
```

```php
<?php

declare(strict_types=1);

require_once 'vendor/autoload.php';

// Factory::create() returns a Faker\Generator wired with default providers.
$faker = Faker\Factory::create();          // or Factory::create('fr_FR') for a locale

echo $faker->name();                        // 'Vince Sporer'
echo $faker->email();                       // 'walter.sophia@hotmail.com'
echo $faker->text();                        // a lorem-ipsum sentence

// Deterministic output: seed once, get the same stream back.
$faker->seed(1234);

// Modifiers wrap any formatter.
$faker->unique()->email();                  // never repeats within this run
$faker->optional()->phoneNumber();          // sometimes returns null
$faker->numberBetween(1, 100);
```

## Architecture / How It Works

Faker is a thin dispatch layer over a collection of **providers**. A
`Faker\Generator` holds an ordered list of provider objects; each provider is a
plain class whose public methods (`name()`, `address()`, `sentence()`, …) are the
"formatters." `Factory::create()` instantiates the `Generator` and pushes the
default set of providers onto it based on the requested locale.

Calls like `$faker->name()` do not hit a real method on `Generator`. They go
through PHP's `__call()` magic, which forwards to `Generator::format($method)`,
which scans the registered providers in reverse order and invokes the first one
that implements a matching formatter. This is why extending Faker is just
`$faker->addProvider($myProvider)` — later providers shadow earlier ones.

**Localization** works by provider stacking. A locale like `fr_FR` loads the
`Faker\Provider\fr_FR\*` classes on top of the base `en_US` providers. If the
locale has no French implementation for a given formatter, the call silently falls
through to the `en_US` provider underneath. There is no error — you get English
data where the locale is incomplete.

**Modifiers** are wrapper proxies returned by methods on the generator:

- `unique()` — remembers every value it has returned and retries until it finds an
  unused one; throws `OverflowException` after ~10,000 collisions.
- `optional($weight)` — returns `null` a fraction of the time.
- `valid($validator)` — retries until a callback approves the value.

Underlying randomness comes from PHP's Mersenne Twister (`mt_rand`), seeded by
`seed()`. This makes runs reproducible but is explicitly **not** cryptographically
secure.

## Production Notes

**Do not use Faker for anything security-sensitive.** Passwords, tokens, and salts
generated from `mt_rand` are predictable, especially once a seed is known. Use
`random_bytes()` / `random_int()` for real entropy.

**`unique()` holds state and can explode.** The uniqueness tracker keeps every
returned value in memory for the life of the generator and throws
`OverflowException` when the value space is too small for the number of rows you
are generating (e.g. `unique()->numberBetween(1, 50)` for 100 records). Reset it
deliberately between logical batches with `$faker->unique($reset = true)` rather
than letting it accumulate across a large seeder.

**Values are not stable across versions.** The backward-compatibility promise
covers data *types*, not data *values*[^3]. A minor upgrade can change what a
seeded call returns, so assertions like `assertEquals('Vince Sporer', $faker->name())`
will break on upgrade. Never snapshot-test raw Faker output.

**Locale coverage is uneven.** Non-English locales range from thorough to nearly
empty, and gaps fall back to `en_US` without warning — so a `ja_JP` fixture set can
quietly contain American addresses. Verify the specific formatters you depend on
for your locale.

**Properties were deprecated in favor of method calls.** Older code used
`$faker->name` (property access); the current API is `$faker->name()` (method
call). The repo ships a Rector configuration to automate the migration:
`vendor/bin/rector process src/ --config vendor/fakerphp/faker/rector-migrate.php`[^2].

**Named arguments are outside the BC promise.** Because PHP 8 named arguments make
parameter names part of the public contract, Faker explicitly excludes argument
names from Semver guarantees — pass positionally to be safe[^3].

**Ship it as a dev dependency.** Faker belongs in `require-dev`. Pulling it into
production runtime bloats the autoloader with data files you do not need at
runtime.

## When to Use / When Not

**Use when:**
- Seeding dev/test databases or building fixtures with plausible-looking data.
- Anonymizing or scrambling data copied from production for lower environments.
- You want a large formatter catalog and locale support without writing generators.
- You are on Laravel/Symfony and want the community-standard the frameworks assume.

**Avoid when:**
- You need cryptographic randomness (tokens, passwords, secrets) — wrong tool.
- You need reproducible, value-stable fixtures across library upgrades — pin exact
  values yourself instead.
- You need statistically representative or referentially-consistent synthetic data
  (real distributions, valid foreign keys) — Faker is per-field random, not
  relational.
- Runtime/production data generation on a hot path — it is convenience-speed, not
  throughput-optimized.

## Alternatives

- fzaninotto/Faker — the original, archived in 2020; use FakerPHP/Faker instead, it
  is the same namespace under active maintenance.
- nelmio/alice — use when you want declarative YAML/PHP fixture *definitions* layered
  on top of Faker rather than writing imperative generation loops.
- league/factory-muffin — use when you want model-factory-style object hydration and
  can tolerate a less actively maintained dependency.
- joke2k/faker — use when your stack is Python; it is the equivalent library there.
- faker-js/faker — use when your stack is JavaScript/TypeScript.

## History

| Version | Date | Notes |
|---------|------|-------|
| fzaninotto/Faker | 2011–2020 | Original library; author archived it in late 2020[^1]. |
| FakerPHP/Faker 1.9.x | 2020 | Community fork begins, namespace- and API-compatible with the archived upstream[^2]. |
| 1.x (ongoing) | 2020– | Property access deprecated for method calls; Rector migration shipped; rolling PHP-version support[^2][^3]. |
| 2.0 (default branch) | in progress | Development target branch on the repo. |

## References

[^1]: François Zaninotto archived `fzaninotto/Faker`; see the archived repository notice. https://github.com/fzaninotto/Faker
[^2]: FakerPHP/Faker README — installation, PHP >= 7.4, Rector migration, MIT license. https://github.com/FakerPHP/Faker
[^3]: FakerPHP/Faker backward-compatibility promise (Semver, data-type-only stability, named-argument exclusion), README. https://github.com/FakerPHP/Faker#backward-compatibility-promise

## Tags

php, fake-data, test-fixtures, database-seeding, data-anonymization, testing, faker, providers, localization, composer-package
