# pestphp/pest

> A closure-based PHP testing framework that wraps PHPUnit with a functional API and an `expect()` assertion layer.

[GitHub repo](https://github.com/pestphp/pest) ·
[Official website](https://pestphp.com) ·
[License: MIT](https://github.com/pestphp/pest/blob/4.x/LICENSE.md)

## Overview

Pest is a testing framework for PHP built and maintained by Nuno Maduro, first released in 2021[^1]. Its premise is that PHPUnit's class-and-method, xUnit-style structure adds ceremony that discourages testing, and that a terser, closure-based syntax — `test('it works', fn () => expect(true)->toBeTrue())` — brings the ergonomics closer to Jest or RSpec. It is now one of the two dominant test runners in the PHP ecosystem, especially in the Laravel community, where it ships as the default in new Laravel skeletons[^2].

The defining fact about Pest is that it is not a replacement for PHPUnit — it is a layer on top of it. Pest transforms its closure-based test files into PHPUnit `TestCase` subclasses at runtime and delegates execution to PHPUnit's runner. This buys enormous compatibility (any PHPUnit assertion, extension, and coverage tooling works) but also welds Pest's release cadence to PHPUnit's: each Pest major tracks a PHPUnit major, and PHPUnit's deprecations become Pest's problems.

The `expect()` API and the closure model are the parts users interact with; the parts they inherit — configuration in `phpunit.xml`, coverage drivers, the underlying assertion semantics — are pure PHPUnit. That split is the source of both Pest's appeal and most of its rough edges.

## Getting Started

```bash
composer require pestphp/pest --dev --with-all-dependencies
./vendor/bin/pest --init      # scaffolds tests/, Pest.php, phpunit.xml
./vendor/bin/pest
```

```php
<?php
// tests/Unit/ExampleTest.php

test('sums two numbers', function () {
    expect(1 + 1)->toBe(2);
});

it('validates emails', function (string $email, bool $valid) {
    expect(filter_var($email, FILTER_VALIDATE_EMAIL) !== false)->toBe($valid);
})->with([
    ['user@example.com', true],
    ['not-an-email', false],
]);
```

`$this` inside a test closure is bound to the underlying PHPUnit `TestCase`, so `$this->get('/')` and other framework helpers work once a base class is wired via `uses(TestCase::class)->in('Feature')` in `tests/Pest.php`.

## Architecture / How It Works

Pest's core trick is source transformation. Test files are plain PHP that call global functions (`test`, `it`, `beforeEach`, `expect`, `uses`), and Pest's autoloader rewrites each file into an anonymous PHPUnit `TestCase` subclass before it is executed. The `Pest\` binary boots a kernel that configures PHPUnit, discovers these files, and hands off to PHPUnit's `TestRunner`. This is why Pest and hand-written PHPUnit test classes can coexist in the same suite and share one `phpunit.xml`.

Key pieces of the model:

- **The expectation API** — `expect($value)->toBe(...)`, `->toBeTrue()`, `->each()`, `->and()`, and higher-order chaining. This is Pest's own fluent layer, distinct from PHPUnit's `assert*` methods (which remain available). Custom expectations are registered with `expect()->extend()`.
- **`uses()` binding** — because closures have no class, the base `TestCase`, traits (e.g. `RefreshDatabase`), and `setUp` logic are attached per-directory via `uses(...)->in(...)`. This indirection is invisible until it breaks.
- **Datasets** — `->with([...])` fans a single test into parameterized cases; named and shared datasets replace PHPUnit data providers.
- **Higher-order tests** — chaining methods directly off `test(...)` (e.g. `->throws()`, `->group()`, `->skip()`) instead of writing a body.
- **Plugins as first-class subsystems** — architecture testing (`arch()` presets), mutation testing, type coverage, snapshot testing, `--watch`, and parallel execution (via ParaTest) are shipped as plugins, some folded into core over time[^3].

Because everything routes through PHPUnit, Pest inherits PHPUnit's coverage (Xdebug or PCOV), its `--filter`/group semantics, and its compatibility matrix — and its constraints.

## Production Notes

**Version coupling is the main upgrade tax.** A Pest major is pinned to a PHPUnit major (Pest 2 → PHPUnit 10, Pest 3 → PHPUnit 11, Pest 4 → PHPUnit 12). You cannot mix and match; upgrading one usually forces the other, and PHPUnit's deprecation waves (metadata attributes replacing doc-comment annotations, stricter data-provider rules) surface through Pest even though you never wrote PHPUnit directly.

**Stack traces and IDE support are weaker than plain PHPUnit.** Since test bodies are closures inside a generated class, failure traces have historically been noisier, and running/debugging a single test from the editor needs the Pest IDE plugin rather than PHPUnit's built-in integration. This has improved but is still a step down from vanilla PHPUnit tooling.

**Parallel testing is ParaTest under the hood.** `--parallel` shells out to separate processes; anything relying on shared in-process state, a single test database, or ordering will break. Laravel users typically need per-process database isolation. Parallelism also disables some coverage configurations depending on driver.

**Global function namespace.** `test`, `it`, `expect`, `uses`, and dataset helpers are global functions. Collisions with application code or other libraries defining the same names are rare but real, and static analyzers need the Pest extension to understand them.

**Coverage and type coverage need drivers.** `--coverage` requires Xdebug or PCOV installed; without one it silently reports nothing useful. Type coverage (`--type-coverage`) is a separate metric measuring annotation completeness, not code execution.

**v4 browser testing raises the dependency floor.** Pest 4's real-browser testing pulls in a Playwright-backed runtime[^4]; CI images need the browser binaries, which changes container size and cold-start characteristics for pipelines that previously only needed PHP and an assertion driver.

## When to Use / When Not

**Use when:**
- You're on Laravel or another modern PHP stack and want the terser default that ships with new projects.
- You value the `expect()` fluent API and closure ergonomics over xUnit classes.
- You want architecture tests, mutation testing, or type coverage without assembling separate tools.

**Avoid when:**
- You want zero abstraction over the runner — every layer Pest adds is a layer to debug when something goes wrong.
- You depend on PHPUnit-native IDE/CI integrations that assume real `TestCase` classes and standard traces.
- You're on a legacy codebase pinned to an old PHPUnit that Pest's current major won't support.

## Alternatives

- sebastianbergmann/phpunit — the engine Pest sits on; use it directly for no magic, canonical traces, and the widest tooling support.
- codeception/codeception — use when you want unit, functional, and acceptance testing unified in one full-stack framework.
- phpspec/phpspec — use when you want spec-driven, example-first BDD design rather than a general runner.
- behat/behat — use when non-developers need Gherkin acceptance scenarios.
- kahlan/kahlan — use when you want describe/it BDD without a PHPUnit dependency at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2021-01 | First public release by Nuno Maduro[^1]. |
| 1.0 | 2021 | First stable; closure API, datasets, expectation layer. |
| 2.0 | 2023-03 | Retargeted PHPUnit 10; expanded plugin ecosystem[^3]. |
| 3.0 | 2024-09 | Mutation testing, type coverage, arch presets; PHPUnit 11[^3]. |
| 4.0 | 2025 | Browser testing, visual regression, smoke testing; PHPUnit 12[^4]. |

## References

[^1]: Pest — official site and documentation. https://pestphp.com
[^2]: Laravel testing docs (Pest as the default test runner in new skeletons). https://laravel.com/docs/testing
[^3]: Pest 3 announcement — mutation testing, type coverage, arch presets. https://pestphp.com/docs/pest3-is-now-available
[^4]: Pest v4 announcement — browser testing. https://pestphp.com/docs/pest-v4-is-here-now-with-browser-testing

## Tags

php, testing, test-framework, phpunit, unit-testing, laravel, bdd, tdd, developer-tools, mutation-testing
