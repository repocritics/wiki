# DiUS/java-faker

> A Java port of Ruby's Faker gem for generating fake placeholder data — widely used, effectively frozen since 2020.

[GitHub repo](https://github.com/DiUS/java-faker) ·
[Official website](http://dius.github.io/java-faker) ·
License: Apache-2.0[^1]

## Overview

java-faker generates realistic-looking fake data — names, addresses, phone numbers, company names, emails, and a long tail of themed generators (Harry Potter, Game of Thrones, Chuck Norris, Pokémon). It is a port of Ruby's `faker` gem (itself descended from Perl's `Data::Faker`), reusing the same YAML fixture files that drive the Ruby project[^2]. The intended use is seeding development databases, populating demos, and supplying non-sensitive values in tests where the exact data does not matter but its shape does.

Maintained by DiUS Computing (an Australian consultancy), the project accumulated broad adoption across the JVM ecosystem — thousands of dependent projects pull `com.github.javafaker:javafaker` from Maven Central. Its defining characteristic in 2026, however, is that it is essentially unmaintained: the last published release is 1.0.2 (2020) and the repository's last commit is mid-2024[^3], while 225 open issues sit unaddressed. The library still works and is still downloaded heavily, but new locales, generators, and bug fixes have stopped landing here.

The practical tension is that java-faker's popularity is inertial. Anyone starting fresh should weigh the actively maintained hard fork, Datafaker, which was created specifically because this repository stalled.

## Getting Started

Maven:

```xml
<dependency>
    <groupId>com.github.javafaker</groupId>
    <artifactId>javafaker</artifactId>
    <version>1.0.2</version>
</dependency>
```

Gradle:

```groovy
dependencies {
    implementation 'com.github.javafaker:javafaker:1.0.2'
}
```

```java
Faker faker = new Faker();

String name    = faker.name().fullName();          // "Miss Samanta Schmidt"
String street  = faker.address().streetAddress();  // "60018 Sawayn Brooks Suite 449"
String company = faker.company().name();           // "Hilll and Sons"

// Deterministic output: seed the underlying RandomService
Faker seeded = new Faker(new Random(42));

// Locale-specific data
Faker german = new Faker(new Locale("de"));
```

## Architecture / How It Works

The core is a thin facade. A `Faker` instance owns a `FakeValuesService` plus a `RandomService`, and each domain accessor (`faker.name()`, `faker.address()`, …) returns a lazily constructed object whose methods resolve string templates against bundled data.

The data itself lives in YAML files under `src/main/resources/<locale>.yml`, copied from the Ruby faker project. Generators do not hard-code values; they call keys like `name.first_name` and let `FakeValuesService` pick a random entry. Two template mechanisms sit on top:

- **`#{...}` interpolation** — resolves another key, allowing composed values (a full name is `#{name.first_name} #{name.last_name}`).
- **`numerify` / `letterify` / `bothify`** — replace `#` with a random digit and `?` with a random letter, so phone-number and postcode patterns expand from templates.

Locale resolution falls back from a specific locale to its language (`en-CA` → `en`) and finally to the English default, so a partially translated locale still produces output. Determinism runs through the single `RandomService`; passing your own `Random` (or seed) makes an entire generation run reproducible, which matters for tests that assert on generated values.

The coupling worth understanding is to the upstream Ruby YAML format. Because java-faker mirrors Ruby faker's fixtures rather than owning them, its data freshness is bounded by whenever someone last synced those files — and that syncing has stopped.

## Production Notes

**It is dormant, not just "stable."** Treat 1.0.2 as the terminal release. Bugs filed against it will not be fixed upstream. The most consequential migration path is to `net.datafaker:datafaker`, a fork that kept the `Faker` API largely source-compatible, added locales and generators, moved to newer Java baselines, and ships regular releases[^4]. Many teams migrate by changing the coordinate and the import package.

**Not cryptographically random and not for security.** The generator is seeded pseudo-randomness intended for readable placeholder data. Do not use it to mint tokens, passwords, or anything requiring unpredictability. `faker.internet().password()` produces test-shaped strings, not secure secrets.

**Uniqueness is not guaranteed.** Repeated calls can and will collide (especially for small domains like first names or booleans). If you seed a database column with a UNIQUE constraint, you must dedupe yourself; java-faker has no built-in unique-value tracking (Datafaker added one).

**Startup and memory cost.** The full set of locale YAML files is bundled in the JAR and parsed via SnakeYAML on first access. For most apps this is negligible, but in memory-constrained or fast-startup contexts (serverless, CLI tools) the reflection- and YAML-heavy initialization is measurable. Reuse a single `Faker` instance rather than constructing per call.

**Transitive dependencies.** Older lines of java-faker pulled in SnakeYAML and Apache Commons versions that have since had CVEs flagged by scanners. Pin and audit; this is another reason the unmaintained status is a real operational cost rather than a cosmetic one.

**Locale gaps.** The advertised locale list is broad, but coverage per generator is uneven — a locale may translate names but fall back to English for addresses or companies. Verify the specific fields you depend on rather than assuming full localization.

## When to Use / When Not

**Use when:**
- You have an existing codebase already on `javafaker` and it meets your needs — there is no urgency to churn.
- You need quick, readable placeholder data for demos, fixtures, or seed scripts and don't need the newest locales or generators.
- You want a well-understood, dependency-light API with abundant Stack Overflow coverage.

**Avoid when:**
- You're starting a new project — prefer the maintained fork, Datafaker.
- You need recent bug fixes, new locales, unique-value constraints, expressions, or CSV/JSON/SQL bulk generation.
- You need any security or uniqueness guarantee from the generated values.
- Your supply-chain policy rejects dependencies with no upstream maintenance and open, unpatched vulnerabilities.

## Alternatives

- datafaker-net/datafaker — the actively maintained hard fork of this project; the default choice for new JVM code needing near-identical API plus ongoing releases.
- faker-ruby/faker — the original Ruby gem this port tracks; use when you are in Ruby rather than the JVM.
- faker-js/faker — the JavaScript/TypeScript equivalent for Node and browser code.
- joke2k/faker — the mature Python equivalent, use in Python test and seeding workflows.
- instancio/instancio — a JVM alternative that populates whole object graphs by reflection rather than field-by-field faker calls; use when you want random valid POJOs, not themed strings.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-06 | Repository created; early port of Ruby faker[^3]. |
| 0.x | 2015–2018 | Long series of 0.x releases; locale and generator growth. |
| 1.0.0 | 2019 | First 1.x release; API stabilization. |
| 1.0.2 | 2020 | Final published release to Maven Central. |
| (fork) | 2022 | Datafaker forked as the maintained successor[^4]. |
| — | 2024-06 | Last commit to the repository; no releases since 1.0.2[^3]. |

## References

[^1]: Repository LICENSE file and README state Apache License 2.0, "Copyright (c) 2019 DiUS Computing Pty Ltd." GitHub's license detector reports the SPDX id as NOASSERTION (unrecognized header), but the license text is Apache-2.0. https://github.com/DiUS/java-faker/blob/master/LICENSE
[^2]: README: "This library is a port of Ruby's faker gem (as well as Perl's Data::Faker library) that generates fake data." https://github.com/DiUS/java-faker/blob/master/README.md
[^3]: GitHub API repository metadata: created 2011-06-06, last push 2024-06-12, 225 open issues, ~4.9k stars, ~862 forks (fetched 2026-07). https://github.com/DiUS/java-faker
[^4]: Datafaker — maintained fork of java-faker. https://www.datafaker.net/ · https://github.com/datafaker-net/datafaker

## Tags

java, jvm, test-data, fake-data, data-generation, faker, testing, fixtures, maven, unmaintained
