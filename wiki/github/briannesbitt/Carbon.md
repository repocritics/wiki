# briannesbitt/Carbon

> A fluent extension of PHP's DateTime with human-readable diffs, localization, and test-time mocking — the de facto date library of the PHP ecosystem.

[GitHub repo](https://github.com/briannesbitt/Carbon) ·
[Official website](https://carbon.nesbot.com/) ·
[License: MIT](https://github.com/briannesbitt/Carbon/blob/master/LICENSE)

## Overview

Carbon is a date/time library by Brian Nesbitt, first published in 2012, that subclasses PHP's native `\DateTime` (and, since Carbon 2, `\DateTimeImmutable`) to add a fluent API on top of the standard library rather than replacing it[^1]. Its reach comes almost entirely from being bundled with Laravel — every Eloquent date cast, `now()` helper, and framework timestamp is a Carbon instance — which makes it one of the most-installed PHP packages in existence (tens of millions of Composer installs per month via `nesbot/carbon`) at ~16.6k GitHub stars.

The library's value is ergonomic, not algorithmic: `->addDays(3)`, `->diffForHumans()` ("2 minutes ago"), `->isWeekend()`, localized `isoFormat()` across 200+ languages, and `Carbon::setTestNow()` to freeze "now" in tests. It leans on `symfony/translation` for its locale data and a `Macroable` trait for runtime extension.

Its defining tension is mutability. Because the default `Carbon` class inherits from the mutable `\DateTime`, methods like `addDay()` change the receiver in place *and* return it — so an aliased instance you thought was a snapshot silently shifts under you. `CarbonImmutable` exists to fix this, but it is opt-in, and the mutable class remains the default that Laravel and most tutorials hand you. The repository is mid-migration from `briannesbitt/Carbon` to the `CarbonPHP/carbon` organization; both mirrors are kept in sync and the Composer package name is unchanged[^2].

## Getting Started

```bash
composer require nesbot/carbon
```

```php
<?php
require 'vendor/autoload.php';

use Carbon\Carbon;
use Carbon\CarbonImmutable;

echo Carbon::now()->toDateTimeString();          // 2026-07-15 09:30:00
echo Carbon::now('America/Vancouver')->hour;     // timezone-aware

// Human diffs and localization
echo Carbon::now()->subMinutes(2)->diffForHumans();              // '2 minutes ago'
echo Carbon::parse('2019-07-23 14:51')->locale('fr_FR')->isoFormat('LLLL');

// Freeze "now" for deterministic tests, then release
Carbon::setTestNow(Carbon::create(2000, 1, 1));
// ... assertions ...
Carbon::setTestNow();

// Immutable variant returns new instances instead of mutating
$start = CarbonImmutable::now();
$later = $start->addHour();       // $start is unchanged
```

## Architecture / How It Works

Carbon is a thin behavioral layer over the SPL date classes. `Carbon extends \DateTime`; `CarbonImmutable extends \DateTimeImmutable`; both implement `CarbonInterface` and share their logic through a large `Carbon\Traits\Date` trait rather than a common base class (PHP has no multiple inheritance, so the two roots are joined by trait, not by parent). This means every native `DateTime` capability — timezones, DST math, `format()` — is inherited for free, and Carbon only adds sugar and safety on top.

Around the two core classes sit `CarbonInterval` (extends `\DateInterval`, adds fluent construction and humanized output), `CarbonPeriod` (an iterable date range, `CarbonPeriod::create('2026-01-01', '1 week', '2026-03-01')`), and `CarbonTimeZone`. Localization is delegated to `symfony/translation` with translation files shipped in the package; `diffForHumans` and `isoFormat` read from that catalog, which is why the dependency footprint is larger than a "just dates" library implies.

Two runtime-global mechanisms are worth understanding. `setTestNow()` installs a process-wide clock that every `Carbon::now()` reads — powerful for tests, but global mutable state that leaks across test cases if not reset. The `Macroable` trait lets any code register methods at runtime (`Carbon::macro('nextPayday', fn () => ...)`), which packages use to extend Carbon globally; this is invisible action-at-a-distance when debugging where a method came from.

## Production Notes

**Mutability is the number-one footgun.** `$a = Carbon::now(); $b = $a; $b->addDay();` moves `$a` too. The idioms that avoid it: use `CarbonImmutable` everywhere (in Laravel, `Date::use(CarbonImmutable::class)`), or call `->copy()` / `->toImmutable()` before mutating a shared instance. Teams that standardize on the immutable class up front hit far fewer date bugs.

**Carbon 2 → 3 changed diff semantics silently.** In Carbon 3 the `diffIn*` methods return a signed `float` (fractional and negative when the argument is in the future) rather than an absolute integer[^3]. Code that did `if ($date->diffInDays($other) > 7)` or relied on integer truncation can change behavior without any error. Audit every `diffIn*` call site on the 2→3 upgrade; use `->diffInDays(absolute: true)` or explicit rounding where you need the old shape.

**`Carbon::parse()` is convenient and dangerous.** It falls back to `strtotime`-style heuristics and will happily misinterpret ambiguous input (e.g. `01/02/2026` day-vs-month) or throw on unparseable strings. For anything from user input or external systems, prefer `Carbon::createFromFormat($fmt, $value)` with an explicit format.

**Version coupling with Laravel.** Because Laravel pins a Carbon range, you rarely upgrade Carbon independently — a major Carbon bump usually arrives with a Laravel major. Trying to force a newer Carbon into an older framework tends to hit peer-dependency conflicts. Plan Carbon upgrades as part of the framework upgrade.

**Performance.** For a few dozen dates per request Carbon's overhead is irrelevant. In hot loops (bulk import, report generation over millions of rows) the localization layer, macro dispatch, and object allocation are measurably heavier than raw `\DateTimeImmutable` or integer timestamps; drop to native types in the tight path.

## When to Use / When Not

**Use when:**
- You're in Laravel or any project that already ships Carbon — it's the ecosystem default.
- You need human-readable diffs, localized formatting across many languages, or test-time clock freezing.
- You want fluent date math without reimplementing DST- and timezone-correct arithmetic.

**Avoid when:**
- You want immutability guaranteed by design with zero footguns and no dependency — reach for native `\DateTimeImmutable` or a stricter library.
- You're on a hot path processing huge volumes of dates where per-object overhead matters.
- You want a strongly-typed model that separates date, time, and instant rather than one `DateTime` blob.

## Alternatives

- cakephp/chronos — Carbon-like fluent API but immutable by default; use it when you want the ergonomics without the mutable-default footgun and with a lighter dependency tree.
- brick/date-time — use when you want a strict, fully immutable model with separate `LocalDate` / `LocalTime` / `Instant` types instead of a single `DateTime` subclass.
- Native \DateTimeImmutable — use when you want zero dependencies and don't need human diffs or localization; the standard library already covers correct timezone/DST math.
- symfony/clock — use when you only need mockable "now" for tests (a `ClockInterface`) rather than a full date library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2012 | Initial release; single mutable `Carbon extends \DateTime`, PHP 5.3+[^1]. |
| 2.0 | 2018 | `CarbonImmutable`, `CarbonInterval`/`CarbonPeriod`, Symfony-based localization, `Macroable`; PHP 7.1+[^4]. |
| 3.0 | 2024 | PHP 8.1+, drops legacy APIs, signed float `diffIn*`, updated Symfony deps[^3]. |

## References

[^1]: Carbon documentation and history. https://carbon.nesbot.com/docs/
[^2]: Repository README migration note, `briannesbitt/Carbon` → `CarbonPHP/carbon` (code kept in sync on both). https://github.com/briannesbitt/Carbon
[^3]: Carbon 3 upgrade guide — `diffIn*` methods now return signed floats. https://carbon.nesbot.com/docs/#api-carbon-3
[^4]: Carbon 2 announcement / changelog. https://carbon.nesbot.com/docs/
[^5]: Composer package `nesbot/carbon` on Packagist. https://packagist.org/packages/nesbot/carbon

## Tags

php, datetime, date-time, localization, laravel, immutability, timezone, developer-tools, library, date-math
