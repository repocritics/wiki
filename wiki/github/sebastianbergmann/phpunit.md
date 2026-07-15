# sebastianbergmann/phpunit

> The de facto unit testing framework for PHP — an xUnit-architecture harness that nearly every PHP project and toolchain builds on top of.

[GitHub repo](https://github.com/sebastianbergmann/phpunit) ·
[Official website](https://phpunit.de/) ·
[License: BSD-3-Clause](https://github.com/sebastianbergmann/phpunit/blob/main/LICENSE)

## Overview

PHPUnit is a programmer-oriented testing framework for PHP, created and still primarily maintained by Sebastian Bergmann[^1]. It is an instance of the xUnit family (Kent Beck's JUnit lineage): tests are methods on classes that extend `TestCase`, assertions are instance methods, and a command-line runner discovers and executes them. It has been the default answer to "how do I test PHP?" for roughly two decades, and its assertion API and test-double system are effectively a shared vocabulary across the ecosystem — Symfony, Laravel, Composer, and most major PHP libraries test with it, and higher-level tools (Pest, Codeception, Paratest) wrap it rather than replace it.

The defining tension is that PHPUnit is a **fast-moving target with a yearly major-version cadence**. A new major ships every February, each carrying a batch of deprecations and removals[^2]. This keeps the framework modern (PHP 8 attributes, a rewritten event system, strict types throughout) but means test suites accumulate deprecation warnings and periodically need mechanical migration. The framework favors correctness and explicitness over backward-compatibility promises, which is a reasonable stance for a test tool but surprises teams that treat it as install-and-forget.

PHPUnit 10 (2023) was the largest internal break in the project's history: the old `TestListener` hook interface was replaced by a typed event system, and PHP 8 attributes (`#[Test]`, `#[DataProvider]`, `#[CoversClass]`) were introduced as the successor to docblock annotations[^3].

## Getting Started

Install per-project with Composer (the recommended path) rather than globally:

```bash
composer require --dev phpunit/phpunit
./vendor/bin/phpunit --version
```

A minimal test using modern attribute metadata and a static data provider:

```php
<?php declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\DataProvider;

final class MathTest extends TestCase
{
    public function testAdditionIsExact(): void
    {
        $this->assertSame(4, 2 + 2);          // strict: type + value
    }

    #[DataProvider('additionCases')]
    public function testAdd(int $a, int $b, int $expected): void
    {
        $this->assertSame($expected, $a + $b);
    }

    public static function additionCases(): array
    {
        return [[0, 0, 0], [1, 1, 2], [2, 3, 5]];
    }
}
```

```bash
./vendor/bin/phpunit tests
```

Configuration lives in `phpunit.xml` (commit `phpunit.xml.dist`, gitignore the local `phpunit.xml` override). It declares test suites, coverage include/exclude paths, bootstrap file, and environment variables.

## Architecture / How It Works

PHPUnit is a monorepo-adjacent project: the `phpunit/phpunit` package is the runner and assertion surface, but much of the real work lives in Sebastian Bergmann's constellation of small single-purpose libraries — `sebastian/comparator` (value equality), `sebastian/diff` (assertion failure diffs), `sebastian/exporter` (value pretty-printing), and `phpunit/php-code-coverage` (coverage)[^1]. Composer stitches them together. This decomposition is deliberate and is why upgrading PHPUnit pulls a fan of transitive `sebastian/*` bumps.

Core moving parts:

- **Test discovery & execution.** The CLI loads the config, builds a test suite tree from your classes, and runs each test method in isolation-of-intent (fresh `TestCase` instance per method). `setUp()`/`tearDown()` bracket each method; `setUpBeforeClass()`/`tearDownAfterClass()` are static and bracket the class.
- **Assertions.** Static and instance assertion methods throw `ExpectationFailedException` on failure. The comparator/diff/exporter stack turns a failed `assertEquals` into a readable diff. The recurring footgun here is `assertEquals` (loose, `==`) versus `assertSame` (strict, `===`) — the former will treat `"1"` and `1` as equal.
- **Test doubles.** `createMock()`, `createStub()`, and `MockBuilder` generate mock subclasses at runtime via code generation. The built-in mocker works by subclassing, so it **cannot** double `final` classes, `static` methods, or `private` methods — a hard architectural limit, not a bug.
- **Metadata.** As of PHPUnit 10 there are two metadata systems: PHP 8 attributes (`#[Test]`, `#[DataProvider]`, `#[CoversClass]`, `#[RunInSeparateProcess]`) and the legacy docblock annotations (`@test`, `@dataProvider`, `@covers`). Attributes are the future; annotations were deprecated in PHPUnit 11[^4].
- **Event system.** Since PHPUnit 10, extensions subscribe to typed events (`Test\Prepared`, `Test\Failed`, `Test\Finished`, etc.) instead of implementing the old listener interface. This is how coverage, logging, and third-party plugins hook in.
- **Coverage.** `php-code-coverage` requires a driver: Xdebug or PCOV (phpdbg support was dropped). It records which lines executed and maps them to `@covers`/`#[CoversClass]` scoping.

## Production Notes

**Coverage needs a driver, and the driver dominates runtime.** Xdebug instruments every line and can slow a suite by 2–5×; PCOV is coverage-only but far faster and is the usual CI choice. Running the suite without coverage in the common `dev` case, and only enabling PCOV in the coverage CI job, is the standard split. Never leave Xdebug loaded in production or in non-coverage test runs.

**Static data providers are mandatory.** Data provider methods must be `static` as of PHPUnit 10 (non-static providers were deprecated in 9 and removed). Providers are executed *before* any test runs, so they cannot depend on `setUp()` state — a frequent source of confusion when migrating.

**Global and static state leaks between tests.** PHPUnit runs all tests in a single process by default. Static properties, singletons, and superglobals mutated by one test are visible to the next. `#[RunInSeparateProcess]` / `#[RunTestsInSeparateProcesses]` fix isolation but fork a new PHP process per test and are very slow; `#[BackupStaticProperties]` and `#[BackupGlobals]` are cheaper but imperfect. Design tests to avoid shared mutable state rather than reaching for process isolation.

**No built-in parallelism.** PHPUnit executes serially. For large suites the community answer is `brianium/paratest`, which shards test files across worker processes. It works well but interacts awkwardly with tests that assume a single shared database or that write to fixed temp paths.

**Upgrading across majors is mechanical but non-trivial.** Each February major removes previously-deprecated APIs. `rector` ships PHPUnit rule sets that automate most attribute migrations and API renames; running Rector before a major bump is the least-painful path. Suites that ignore deprecation warnings for several years face a large one-time migration.

**Deprecation noise is signal.** PHPUnit is aggressive about surfacing deprecations (both its own and PHP's) in test output. Teams often suppress them with `failOnDeprecation="false"` or display settings in `phpunit.xml`; the better move is to treat the deprecation list as the upgrade to-do list.

## When to Use / When Not

**Use when:**
- You are writing any PHP project and need unit, integration, or feature tests — this is the ecosystem default and what every framework's test helpers assume underneath.
- You want a mature assertion API, test doubles, and coverage integration in one tool.
- You need CI-grade output formats (JUnit XML, TestDox, TeamCity) that downstream tooling already understands.

**Avoid / reach elsewhere when:**
- You want expressive, low-ceremony test syntax without class boilerplate — Pest sits on top of PHPUnit and gives that without giving up the engine.
- You need full-stack acceptance/browser/API testing orchestration — Codeception or Behat cover that layer (and still delegate unit tests to PHPUnit).
- You must mock `final`/`static`/`private` members heavily — the built-in mocker cannot, and fighting it signals a design or tooling mismatch.

## Alternatives

- pestphp/pest — a testing framework built *on* PHPUnit; closure-based `it()`/`expect()` syntax. Use instead when you want expressive tests but keep PHPUnit's engine and ecosystem.
- Codeception/Codeception — full-stack (unit/functional/acceptance) suite that wraps PHPUnit. Use when you need browser and API acceptance testing in one tool.
- phpspec/phpspec — SpecBDD, design-by-specification. Use when you want tests to drive object design rather than verify existing code.
- behat/behat — Gherkin BDD for human-readable acceptance scenarios. Use for stakeholder-facing behavior specs, not unit testing.
- infection/infection — mutation testing that *complements* PHPUnit rather than replacing it. Use alongside to measure whether your PHPUnit tests actually catch regressions.

## History

| Version | Date | Notes |
|---------|------|-------|
| 6.0 | 2017-02 | Namespaced classes, PHP 7 required. |
| 7.0 | 2018-02 | Void return types, PHP 7.1+. |
| 8.0 | 2019-02 | `void` on `setUp`/`tearDown`, stricter signatures. |
| 9.0 | 2020-02 | Non-static data providers deprecated; PHP 7.3+. |
| 10.0 | 2023-02 | Event system replaces `TestListener`; PHP 8 attributes introduced[^3]. |
| 11.0 | 2024-02 | Docblock annotation metadata deprecated in favor of attributes[^4]. |
| 12.0 | 2025-02 | Continued removals; PHP 8.3+ baseline[^2]. |

## References

[^1]: Sebastian Bergmann, PHPUnit and related components. https://sebastian-bergmann.de/open-source.html
[^2]: PHPUnit follows a yearly major-version cadence with a supported-versions/lifecycle policy. https://phpunit.de/supported-versions.html
[^3]: PHPUnit 10 announcement and changelog (event system, attributes). https://phpunit.de/announcements/phpunit-10.html
[^4]: PHPUnit ChangeLog — deprecation of metadata in doc-comments. https://github.com/sebastianbergmann/phpunit/blob/main/ChangeLog-11.md

## Tags

php, testing, unit-testing, xunit, test-framework, code-coverage, mocking, phpunit, composer, tdd
