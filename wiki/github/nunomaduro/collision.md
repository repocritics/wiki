# nunomaduro/collision

> Terminal-facing error reporting for PHP CLI apps and test runners — the pretty stack traces you see when a Laravel command or a Pest/PHPUnit suite fails.

[GitHub repo](https://github.com/nunomaduro/collision) ·
[Packagist](https://packagist.org/packages/nunomaduro/collision) ·
[License: MIT](https://github.com/nunomaduro/collision/blob/v8.x/LICENSE.md)

## Overview

Collision is a `--dev` Composer package that intercepts uncaught exceptions and errors thrown on the command line and renders them as syntax-highlighted, human-readable reports: the message, a code excerpt around the failing line, a trimmed stack trace, and (where available) a suggested solution. It is built on top of `filp/whoops`[^1] and targets the console specifically — it is not an HTTP error page renderer. Created and maintained by Nuno Maduro since 2017[^2], it is bundled with every Laravel install, which is why most PHP developers encounter it without ever adding it deliberately.

Its second, larger role is as the failure formatter for test runners. Collision ships adapters that hook PHPUnit's and Pest's output, replacing the default dot-and-stack-trace report with the same code-excerpt-and-diff presentation. Pest depends on it directly, so in practice Collision's rendering is what a large share of the PHP test-writing world reads every day.

The defining tension is version lockstep. Collision sits at the intersection of three fast-moving dependencies — Laravel, PHPUnit, and Pest — and pins fairly narrow constraints against all of them. That keeps the output correct across their internals, but it also means a Collision major is effectively gated on those ecosystems, and a premature `composer require phpunit/phpunit:^13` can be blocked until the matching Collision lands. The compatibility matrix in the README is not documentation courtesy; it is the actual upgrade contract.[^3]

## Getting Started

```bash
composer require nunomaduro/collision --dev
```

Requires PHP 8.2+ on the current (v8.x) line.[^3] Under Laravel it is wired up automatically. Outside Laravel, there is no auto-discovery adapter — you register the handler yourself:

```php
<?php
require __DIR__.'/vendor/autoload.php';

(new \NunoMaduro\Collision\Provider)->register();

// From here on, any uncaught Throwable in this CLI process is
// rendered by Collision instead of PHP's default fatal output.
throw new \RuntimeException('Something went wrong');
```

For the test-runner experience, no code is needed: on Laravel/Pest projects the printer is registered through the framework; standalone PHPUnit setups enable it through the framework's own configuration rather than a direct API call.

## Architecture / How It Works

Collision is a thin, opinionated presentation layer stacked on Whoops. The pipeline is roughly: a registered handler catches the `Throwable`, a `Writer` walks the exception and its trace, a highlighter renders the relevant source excerpt with ANSI colors, and an `ArgumentFormatter` summarizes call arguments in frames. Output goes through a Symfony Console `OutputInterface`, which is where the color, width, and verbosity behavior comes from.

Two integration surfaces matter:

- **Application handler.** `Provider::register()` installs Collision as the process error/exception handler. In Laravel this is done for you inside the console kernel, and only for the console/testing context — the web request lifecycle keeps Laravel's own (Ignition-based) renderer. This separation is deliberate: Collision is CLI-only by design.
- **Test printer adapters.** Under `NunoMaduro\Collision\Adapters`, Collision provides the glue that turns PHPUnit/Pest failure events into its report format. This adapter is the most fragile part of the codebase because it binds to test-framework internals rather than a stable public API.

Suggested solutions come through a solution-provider contract (the same "solutions" concept popularized by Ignition): a failing exception can carry recommended fixes that Collision prints beneath the trace. This is optional and mostly surfaces for well-known framework exceptions.

The coupling story is the whole story. Collision has almost no independent surface area of its own; nearly every hard part of the code exists to track someone else's internals — Whoops for trace handling, Symfony Console for rendering, and PHPUnit/Pest for test events. That is why the package is small yet needs frequent major releases.

## Production Notes

- **Keep it in `require-dev`.** Collision registers global error handlers and prints source excerpts and argument values — exactly what you do not want reachable in production. It is a development/testing tool; the `--dev` flag in the install command is not optional advice. Laravel gates its registration to console/testing for the same reason.
- **The PHPUnit 9 → 10 break.** PHPUnit 10 replaced its result-printer mechanism with an event-subscriber system, and the test-adapter had to be rewritten to match. Historically this is the change that most often stranded teams: a project on an older Collision could not move to PHPUnit 10 until it also bumped Collision (and, transitively, often Laravel and Pest). Read the matrix before bumping any one of the four.[^3]
- **Composer conflict, not runtime failure.** When versions do not line up, the symptom is a `composer update` that refuses to resolve, not a broken test run. The fix is almost always aligning the Laravel/PHPUnit/Pest/Collision quadruple to a compatible row, not loosening Collision's constraint.
- **Terminal-dependent rendering.** Colors, width, and Unicode box characters depend on the underlying terminal and Symfony Console's TTY detection. CI logs, redirected output, and some Windows shells degrade the presentation; behavior there is a function of the console layer, not Collision itself.
- **Not a web tool.** For rendered exception pages in the browser you want Whoops or Ignition; Collision has nothing to offer the HTTP path.

## When to Use / When Not

**Use when:**
- You want readable CLI failures for a Laravel/Symfony console app or an Artisan-style command runner.
- You want the code-excerpt-and-diff test report for PHPUnit or Pest.
- You are on Laravel or Pest already — it is present and you should leave it in.

**Avoid when:**
- You need error rendering for HTTP responses (use Ignition/Whoops).
- You are shipping a production runtime dependency — it belongs in `require-dev` only.
- You want a rendering layer decoupled from Laravel/PHPUnit/Pest release cadence; the tight version pinning is intrinsic.

## Alternatives

- filp/whoops — the layer Collision is built on; use it directly when you need web error pages or lower-level control over trace handling.
- spatie/laravel-ignition — richer, solution-driven error page for Laravel web requests; complements rather than replaces Collision's CLI role.
- symfony/error-handler — Symfony's built-in exception rendering for both CLI and web; use it when you are Symfony-first and want fewer extra dependencies.
- sebastianbergmann/phpunit — its stock result output is the fallback when you want the test failure report with zero added packages.
- pestphp/pest — if you want the polished test experience out of the box; it depends on Collision, so this is adopting Collision, not avoiding it.

## History

Collision's major line tracks Laravel's, with the PHPUnit/Pest support envelope widening each release. Exact per-major release dates are best confirmed on Packagist/GitHub releases; the version-to-framework mapping below is from the project's own compatibility matrix.[^3]

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2017 | Initial release; CLI exception reporting on top of Whoops.[^2] |
| 3.x | ~2019 | Laravel 6 line. |
| 4.x | ~2020 | Laravel 7 line. |
| 5.x | ~2020 | Laravel 8 line. |
| 6.x | ~2022 | Laravel 9/10; PHPUnit 9, Pest 1. |
| 7.x | ~2023 | Laravel 10; PHPUnit 10, Pest 2 — test-adapter rewrite for PHPUnit's event system. |
| 8.x | current | Laravel 11/12; PHPUnit 10–13, Pest 2–5; PHP 8.2+.[^3] |

## References

[^1]: Whoops — PHP error handler that Collision builds on. https://github.com/filp/whoops
[^2]: nunomaduro/collision README — authorship, Laravel inclusion, and Whoops/Symfony/PHPUnit support. https://github.com/nunomaduro/collision
[^3]: Version Compatibility matrix (Laravel / Collision / PHPUnit / Pest) and PHP 8.2+ requirement, README on the v8.x branch. https://github.com/nunomaduro/collision/blob/v8.x/README.md

## Tags

php, cli, error-reporting, exceptions, laravel, phpunit, pest, testing, console, developer-tools, whoops
