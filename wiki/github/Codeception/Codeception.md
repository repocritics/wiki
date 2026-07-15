# Codeception/Codeception

> Full-stack PHP testing framework that unifies acceptance, functional, and unit tests behind a single readable actor-based API.

[GitHub repo](https://github.com/Codeception/Codeception) ·
[Official website](https://codeception.com) ·
[License: MIT](https://github.com/Codeception/Codeception/blob/main/LICENSE)

## Overview

Codeception is a PHP testing framework created by Michael Bodnarchuk ("davert") with first commits in 2011[^1]. Its premise is that one tool should cover the full testing pyramid — browser-driven acceptance tests, framework-level functional tests, and plain unit tests — using a consistent, English-like DSL rather than three separate toolchains. Tests read as a sequence of actions performed by an "actor" object (`$I->amOnPage('/')`, `$I->click('Login')`, `$I->see('Welcome')`), which is where much of the appeal, and much of the criticism, comes from.

Under the hood it is not a from-scratch test runner. Codeception is an orchestration layer built on top of PHPUnit (assertions, unit test execution), Symfony BrowserKit and the DomCrawler (functional tests), and the WebDriver protocol / Selenium (real-browser acceptance tests)[^2]. The framework's value is the "Module" and "Actor" abstraction that hides those backends behind one grammar. The recurring tension is exactly that abstraction: the magic `$I` actor and the module autoloading are pleasant to read but obscure what is actually executing, and the tight coupling to PHPUnit internals means Codeception releases must track PHPUnit's own release cadence closely.

The audience is teams that want end-to-end and integration coverage of a web application — especially on Symfony, Laravel, Yii, or WordPress, all of which have first-party or well-maintained modules — without assembling Selenium, BrowserKit, and a unit runner by hand. Since around 2020 the wider PHP community has drifted toward Pest and vanilla PHPUnit for unit work, so Codeception's center of gravity has narrowed toward acceptance and functional/API testing where its module ecosystem still has few equals.

## Getting Started

```bash
composer require "codeception/codeception" --dev
php vendor/bin/codecept bootstrap
```

`bootstrap` scaffolds a `tests/` directory with `Unit`, `Functional`, and `Acceptance` suites plus their `.suite.yml` config and generated Actor classes. A minimal acceptance test in the Codeception 5 layout:

```php
<?php
// tests/Acceptance/SignInCest.php
namespace Tests\Acceptance;

use Tests\Support\AcceptanceTester;

class SignInCest
{
    public function signInSuccessfully(AcceptanceTester $I): void
    {
        $I->amOnPage('/login');
        $I->fillField('email', 'user@example.com');
        $I->fillField('password', 'secret');
        $I->click('Sign in');
        $I->see('Welcome');
        $I->seeInCurrentUrl('/dashboard');
    }
}
```

```bash
php vendor/bin/codecept run Acceptance
php vendor/bin/codecept run Acceptance SignInCest:signInSuccessfully --steps
```

## Architecture / How It Works

The core concepts are **Suites**, **Modules**, and **Actors**.

- **Suites** are independently configured groups of tests (`Acceptance.suite.yml`, `Functional.suite.yml`, etc.). Each suite declares which modules it enables. This is why the same `$I->see()` call can drive a real browser in one suite and a headless BrowserKit request in another — the verb is the same, the module behind it differs.
- **Modules** are the actual implementations: `WebDriver` (Selenium/Chrome via the W3C WebDriver protocol), `PhpBrowser` (Guzzle-based headless HTTP), `Symfony`/`Laravel`/`Yii2` (in-memory framework kernel calls, no HTTP), `Db` (fixture seeding and DB assertions), `REST`, `Asserts`, and dozens more. Enabled modules contribute their public methods into the Actor.
- **Actors** (`AcceptanceTester`, `FunctionalTester`, `UnitTester`) are generated classes. Codeception scans the enabled modules and code-generates the actor's method signatures into `_generated/` so IDEs get autocomplete. The `$I` variable is an instance of that generated class; every action you call is proxied to a module.

Test formats: **Cest** (class with public test methods, the modern default), **Cept** (flat procedural scripts, legacy), and **Test/Unit** (PHPUnit-style `extends \Codeception\Test\Unit`). Gherkin `.feature` files are supported for BDD-style specs mapped to step definitions.

The functional-vs-acceptance distinction is central and easy to misuse: functional tests call the framework kernel directly in the same PHP process (fast, but sensitive to global state leaking between requests), while acceptance tests go through a real HTTP server and browser (slow, but true black-box). The same test code can often run in either mode, which is the selling point and also a source of confusing failures when a test depends on in-process behavior that a real browser does not reproduce.

## Production Notes

- **PHPUnit coupling is the dominant upgrade hazard.** Because Codeception wraps PHPUnit's runner and assertion internals rather than only its public API, a new major PHPUnit release frequently cannot be adopted until Codeception ships matching support. Pin both and upgrade them together; expect Codeception's supported PHPUnit range to lag the newest PHPUnit by a release.
- **The v4 module split still bites newcomers.** In Codeception 4 the modules were extracted from the monolith into separate Composer packages (`codeception/module-webdriver`, `codeception/module-db`, `codeception/module-asserts`, etc.)[^3]. Installing `codeception/codeception` alone no longer gives you WebDriver or Db — a missing-module error at runtime almost always means the module package was never required. Version constraints between the core and each module package must stay compatible.
- **WebDriver flakiness is inherited, not solved.** Acceptance tests depend on Selenium/ChromeDriver and a running server; they carry the usual timing, wait, and browser-version fragility. Prefer explicit `$I->waitForElement()` over implicit sleeps, keep ChromeDriver in lockstep with the browser, and run acceptance suites separately from fast functional/unit suites in CI.
- **Functional tests leak state.** Running the framework in-process across many tests can accumulate container/state between requests in ways a real request boundary would not. Symptoms are order-dependent failures. The `Db` module's `cleanup` (transaction rollback per test) and careful fixture reset mitigate this, but not every backend supports it.
- **Startup/discovery cost.** The actor code-generation, module bootstrapping, and YAML suite configuration add overhead and indirection compared to a bare PHPUnit setup. For pure unit testing this weight is rarely worth it — teams increasingly reserve Codeception for the acceptance/functional/API layer and run unit tests directly on PHPUnit or Pest.
- **Configuration lives in YAML, not PHP.** Suite behavior is spread across `codeception.yml` plus per-suite `*.suite.yml`; debugging a misbehaving test often means reading the YAML to see which modules and settings are active, not just the test file.

## When to Use / When Not

**Use when:**
- You need real end-to-end / acceptance coverage of a web app and want browser and headless drivers behind one API.
- You are on Symfony, Laravel, Yii, or WordPress and can lean on a maintained first-party module.
- You want API/REST tests with rich request/response assertions and DB seeding in the same suite.
- Readable, action-oriented test narratives matter to your team more than minimal indirection.

**Avoid when:**
- You only write unit tests — vanilla PHPUnit or Pest is lighter and better supported for that layer.
- You want to stay on the bleeding edge of PHPUnit releases the day they drop.
- You prefer explicit, transparent test code over an actor/module abstraction and code-generated APIs.
- Your team already standardized on Pest and wants one expressive DSL across all test types.

## Alternatives

- sebastianbergmann/phpunit — the unit-testing foundation Codeception sits on; use it directly when you only need unit/integration tests without the actor/module layer.
- pestphp/pest — expressive, closure-based test DSL over PHPUnit; use when you want elegant syntax and are mostly doing unit/feature tests.
- Behat/Behat — pure Gherkin BDD with step definitions; use when business-readable `.feature` specs are the point rather than a coding API.
- laravel/dusk — Laravel-native browser testing over ChromeDriver; use when you are all-in on Laravel and want its expressive browser assertions.
- symfony/panther — real-browser E2E for Symfony using WebDriver and BrowserKit; use when you want a Symfony-native scraper/testing client without Codeception's suite machinery.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | First stable release; Cept scripts, actor DSL, PHPUnit-backed[^1]. |
| 2.0 | 2014 | Cest format, reworked module system, Guy/actor generation. |
| 3.0 | 2019 | Modernized internals, dependency and performance changes. |
| 4.0 | 2020 | Modules extracted into standalone Composer packages[^3]. |
| 5.0 | 2022-10 | PHP 8.0+ required, PHPUnit-modern support, namespaced default layout[^4]. |

## References

[^1]: Codeception GitHub repository history; project created 2011, first stable line 2012. https://github.com/Codeception/Codeception
[^2]: Codeception documentation, "Introduction" — describes PHPUnit, BrowserKit/DomCrawler, and WebDriver as the underlying backends. https://codeception.com/docs/Introduction
[^3]: Codeception 4.0 upgrade notes — modules moved to separate packages such as `codeception/module-webdriver`. https://codeception.com/for/packages
[^4]: Codeception 5 changelog / release. https://github.com/Codeception/Codeception/releases

## Tags

php, testing, acceptance-testing, functional-testing, end-to-end, bdd, phpunit, webdriver, integration-testing, test-framework
