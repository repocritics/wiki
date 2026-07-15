# bchavez/Bogus

> A fake data generator for .NET — a C# port of the JavaScript faker.js with a FluentValidation-style rule API.

[GitHub repo](https://github.com/bchavez/Bogus) ·
[NuGet](https://www.nuget.org/packages/Bogus/) ·
[License: MIT](https://github.com/bchavez/Bogus/blob/master/LICENSE.md)

## Overview

Bogus generates realistic fake data — names, addresses, emails, credit card
numbers, lorem ipsum, dates — for .NET languages (C#, F#, VB.NET). It began in
2015 as a straight port of Marak Squires' `faker.js`, reusing that project's
locale data files, and borrowed its fluent builder syntax from Jeremy Skinner's
FluentValidation[^1]. The primary use case is seeding databases and populating
UI/test fixtures with plausible-looking data instead of `"test1"`, `"test2"`.

The library's center of gravity is `Faker<T>`: you declare a rule per property
(`RuleFor(x => x.Email, f => f.Internet.Email())`) and call `Generate()` to
materialize instances. Rules are lambdas evaluated lazily at generation time, so
one rule can reference already-generated properties of the same object. Beneath
that sits a set of locale-aware `DataSet` classes (`Name`, `Address`,
`Internet`, `Finance`, and so on) that can also be used standalone without any
fluent setup.

The defining tension is determinism versus convenience. Bogus is built around a
single global seed (`Randomizer.Seed`) so test runs can be reproducible, but that
seed is process-global mutable state, and the exact bytes produced for a given
seed are *not* contractually stable across library versions — locale data updates
change outputs. Bogus is excellent for "give me something that looks like a user"
and a poor fit for "give me the exact same value forever." The project is
actively maintained (releases through late 2025) and now ships a paid **Bogus
Premium** tier of extension packages alongside the MIT-licensed core[^2].

## Getting Started

```powershell
Install-Package Bogus
```
```bash
dotnet add package Bogus
```

Minimum target: .NET Standard 1.3 / 2.0, or .NET Framework 4.0.

```csharp
using Bogus;

// Global seed → reproducible runs (see Production Notes for the caveat).
Randomizer.Seed = new Random(8675309);

var userFaker = new Faker<User>()
    .RuleFor(u => u.FirstName, f => f.Name.FirstName())
    .RuleFor(u => u.LastName,  f => f.Name.LastName())
    // Compound rule referencing earlier-generated values:
    .RuleFor(u => u.Email, (f, u) => f.Internet.Email(u.FirstName, u.LastName))
    .RuleFor(u => u.Ssn,   f => f.Random.Replace("###-##-####"));

User user = userFaker.Generate();
List<User> batch = userFaker.Generate(1000);
```

For one-off values without a builder, use the `Faker` facade directly:
`var f = new Faker("en"); var city = f.Address.City();`.

## Architecture / How It Works

Three layers stack on top of each other:

- **`Randomizer`** — wraps a `System.Random` (or a supplied one) and is the
  single source of entropy. Every `Faker` instance derives its randomizer from
  the global `Randomizer.Seed` at construction time unless given a local seed.
- **`DataSet` classes** — `Name`, `Address`, `Internet`, `Commerce`, `Finance`,
  `Lorem`, `Date`, `Company`, `Hacker`, `Vehicle`, etc. Each reads locale
  resource files (the ported faker.js JSON) and exposes methods like
  `FirstName()` or `CreditCardNumber()`. Some methods do real work beyond
  lookups: credit-card numbers carry a valid Luhn checksum, IBAN/routing numbers
  carry valid check digits.
- **`Faker<T>`** — the fluent orchestrator. It stores an ordered list of
  `RuleFor` actions plus optional `CustomInstantiator`, `StrictMode`,
  `RuleForType`, `Ignore`, and `FinishWith` hooks, and applies them when you call
  `Generate`, `GenerateLazy` (deferred `IEnumerable`), or `GenerateForever`.

Locales are selected by string code (`"en"`, `"ko"`, `"de_AT"`, …). When a
locale lacks a given data set, Bogus silently falls back to `en` for that set
only[^3] — so a Korean faker can still emit English lorem if the `ko` lorem set
is missing. Extension methods live in `Bogus.Extensions` and its regional
sub-namespaces (e.g. `Bogus.Extensions.Denmark` for a valid CPR number,
`Bogus.Extensions.UnitedStates` for a valid SSN), which must be imported
explicitly. `OrNull(f, probability)` and `OrDefault` are the common nullable
helpers.

## Production Notes

- **`Randomizer.Seed` is global mutable static state and is not thread-safe.**
  Parallel test frameworks (xUnit runs test classes concurrently by default)
  that share the global seed will interleave calls into one `Random` and produce
  non-reproducible, occasionally colliding output. For isolated determinism give
  each `Faker<T>` its own seed via `.UseSeed(n)` rather than relying on the
  global, and avoid mutating `Randomizer.Seed` mid-run.
- **`StrictMode` is `false` by default.** Any property without a rule is left at
  its CLR default (null / 0) with no warning. Turn on `StrictMode(true)` (or
  `Faker.DefaultStrictMode`) in tests to fail fast when a new model property
  isn't covered.
- **Output is not stable across versions.** A locale data update or generator
  change can alter the value produced for a fixed seed. Do not snapshot/golden
  test against literal Bogus output and then upgrade the package — the diffs will
  be spurious. Assert on shape (non-null, matches regex), not exact strings.
- **`CustomInstantiator` is required for types without a public parameterless
  constructor.** Bogus uses `Activator.CreateInstance` by default; records and
  DI-constructed entities need an explicit instantiator.
- **Performance is reflection-per-property.** This is fine for thousands of rows
  in a seed script; generating millions in a hot path benefits from
  `GenerateLazy`/`GenerateForever` streaming and from reusing one `Faker<T>`
  instance rather than reconstructing it per row.
- **External URLs rot.** `Internet.Avatar()` and some image helpers historically
  returned links to third-party services (uifaces, placeholder hosts) that have
  gone offline over the years; treat generated URLs as strings, not live assets.
- **Premium/OSS split.** The core is MIT, but some newer capabilities are gated
  behind the commercial Bogus Premium license[^2]; check which namespace a
  feature lives in before assuming it's free.

## When to Use / When Not

**Use when:**
- Seeding a database or EF Core context with human-plausible data.
- Populating UI mockups, demos, or load-test payloads.
- You want per-property control and locale support with a readable fluent API.

**Avoid when:**
- You need byte-stable output pinned forever (Bogus makes no cross-version
  guarantee — pin the package version and even then prefer shape assertions).
- You want property-based testing with shrinking — use FsCheck, not Bogus.
- You only need "some anonymous value, I don't care what it looks like" — a
  minimal auto-fixture is lighter than modeling realistic data.

## Alternatives

- AutoFixture/AutoFixture — auto-generates anonymous test values and wires into
  xUnit/NUnit; use when you want objects filled automatically and don't care that
  the data looks realistic.
- fscheck/FsCheck — property-based testing with generators and shrinking; use
  when you're testing invariants over random inputs rather than seeding data.
- faker-js/faker — the JavaScript original Bogus is ported from; use in
  Node/TypeScript projects.
- MelbourneDeveloper/GenFu — older .NET intelligent test-data library; use when
  you want convention-based filling with less rule ceremony.
- nbuilderproject/nbuilder — fluent object builder focused on list generation and
  sequential values; use when you need incrementing/patterned data more than
  realism.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-06 | First commit; port of faker.js with FluentValidation-style API[^1]. |
| v34.x | 2023 | Multi-target modernization; older TFMs pruned over the v33–34 line. |
| v35.0 | 2024 | Current major line; ongoing locale and data-set additions. |
| v35.6.0 | 2024-07-19 | Release on the v35.6 series[^4]. |
| v35.6.5 | 2025-10-26 | Latest release as of this writing[^4]. |

At the time of writing the repo has ~9,700 stars, ~540 forks, and ~90 open
issues, with commits into late December 2025 — a mature, single-maintainer
project (Brian Chavez) that ships small, frequent patch releases.

## References

[^1]: Bogus README — "fundamentally a C# port of faker.js and inspired by FluentValidation's syntax sugar." https://github.com/bchavez/Bogus
[^2]: Bogus Premium Extensions — commercial license tier alongside the MIT core. https://github.com/bchavez/Bogus#bogus-premium-extensions
[^3]: Bogus README, Locales section — "Bogus will default to `en` if a locale-specific data set is not found." https://github.com/bchavez/Bogus
[^4]: Bogus GitHub Releases. https://github.com/bchavez/Bogus/releases

## Tags

csharp, dotnet, fsharp, vb-net, test-data, fake-data, data-generator, faker, seeding, testing, fluent-api, nuget
