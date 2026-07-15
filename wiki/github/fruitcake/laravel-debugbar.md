# fruitcake/laravel-debugbar

> A development-only debug toolbar for Laravel that wraps PHP Debug Bar and adds framework-aware collectors for queries, routes, views, and events.

[GitHub repo](https://github.com/fruitcake/laravel-debugbar) ·
[Official website](http://laraveldebugbar.com/) ·
[License: MIT](https://github.com/fruitcake/laravel-debugbar/blob/master/LICENSE)

## Overview

Laravel Debugbar integrates the framework-agnostic [PHP Debug Bar](https://github.com/php-debugbar/php-debugbar) library with Laravel[^1]. It ships a `ServiceProvider` that registers the debugbar in the container, bootstraps a set of Laravel-specific `DataCollector`s, and injects the rendered toolbar into HTML responses just before the closing `</body>` tag. The result is an in-page panel showing executed SQL queries (with bindings and timing), matched route, rendered views, fired events, log messages, request/response detail, timeline measurements, and memory usage.

The package was authored by Barry vd. Heuvel (`barryvdh`) and is one of the most-installed development dependencies in the Laravel ecosystem. As of 2023 the canonical package name moved from `barryvdh/laravel-debugbar` to `fruitcake/laravel-debugbar`; the old Composer name still resolves but new installs should use the Fruitcake name[^2]. The GitHub repository redirects accordingly.

The defining tension is that Debugbar is deliberately invasive. To show you what happened during a request it registers listeners across the query, view, event, mail, cache, and log subsystems, buffers the collected data, and rewrites the HTML response. That instrumentation both slows the application and, by design, retains and exposes request internals. The README carries an explicit caution: never run it on a publicly accessible site, because stored requests leak information[^1]. It is a local-development instrument, not an APM.

## Getting Started

Require it as a dev-only dependency so it never ships to production:

```shell
composer require fruitcake/laravel-debugbar --dev
```

Laravel package auto-discovery registers the ServiceProvider automatically. The bar activates when `APP_DEBUG=true` and the environment is not `production` or `testing`. To publish the config:

```shell
php artisan vendor:publish --provider="Fruitcake\LaravelDebugbar\ServiceProvider"
```

The `Debugbar` facade lets you push messages, time operations, and record exceptions:

```php
Debugbar::info($user);
Debugbar::warning('Cache miss on key: ' . $key);

Debugbar::measure('Expensive report', function () {
    return generateMonthlyReport();
});

try {
    doRiskyThing();
} catch (\Throwable $e) {
    Debugbar::addThrowable($e);
}
```

Helper functions (`debug(...)`, `debugbar()`) and a `Collection::debug()` macro are also available.

## Architecture / How It Works

Debugbar is a thin Laravel adapter over the upstream `php-debugbar` library, which supplies the collector abstraction, the JavaScript renderer, and the storage/OpenHandler mechanism. This package contributes the Laravel wiring and custom collectors.

At boot the ServiceProvider attaches collectors that subscribe to Laravel's events. The custom collectors include `QueryCollector` (hooks the DB connection to log every statement, its bindings, and duration), `RouteCollector`, `ViewCollector`, `EventsCollector`, a `SymfonyRequestCollector` that replaces the generic request collector with richer request/response data, plus opt-in collectors for logs, files, config, and cache. Laravel's own log and mail systems are bridged through `LogCollector` and `SymfonyMailCollector`. On top sit the upstream defaults: PHP info, messages, timing, memory, and exceptions.

Rendering happens in a terminating step. After the response is generated, a middleware/kernel hook takes the `JavascriptRenderer` output and string-inserts it before `</body>`. Because request data is only complete after the response, disabling auto-inject (to place the renderer manually) also drops the request collector unless you re-enable a `default_request` collector in config. The bar can bundle its own vendor assets (FontAwesome, highlight.js, jQuery) or defer to assets your app already loads.

Persistence is handled by php-debugbar's storage layer: enabling `debugbar.storage.open` writes each request's captured data to disk so the "Browse" button can replay previous requests. Access to that stored history can be gated with a callback receiving the `Request`.

Runtime toggling (`Debugbar::enable()` / `disable()`) exists, but enabling adds collectors and their overhead, so it is not a free switch.

## Production Notes

**Do not deploy it enabled.** This is the headline operator caveat, repeated by the maintainer. Stored-request browsing and the collectors expose queries (including bindings), config values, session data, and environment detail. `--dev` install plus the `production`/`testing` environment guard are the intended defenses; `DEBUGBAR_FORCE_ALLOW_ENABLE` exists but only bootstraps the provider, and using it in production is discouraged outright.

**Measurable overhead.** Collecting and rendering data costs time and memory. When a local app feels slow, the standard first move is to disable heavy collectors (queries with backtraces, views, events). The QueryCollector's optional source-location backtrace is especially expensive on query-heavy pages.

**Long-lived workers need version 4.x.** Under Laravel Octane (Swoole/RoadRunner) the app process persists across requests, so a debugbar that accumulates state would leak memory or bleed one request's data into the next. Debugbar 4.x supports Octane out of the box; if you are upgrading from 3.x you must remove the Debugbar `flush` entry that older setups added to `config/octane.php`[^1].

**AJAX / Livewire / redirects.** The bar handles these via a dropdown of recent requests rather than an inline panel, since there is no fresh full-page HTML to inject into. This works but means the data model is "last N requests," which interacts with storage settings.

**Manual injection breaks silently.** Turning off `inject` without wiring the renderer (and re-adding a request collector) yields a bar with missing request information rather than an error — a common source of confusion.

**Asset conflicts.** If your app already loads jQuery or highlight.js, leaving Debugbar's bundled vendors on can double-load them; the include-vendors config accepts `true`, `false`, `'js'`, or `'css'` to reconcile this.

## When to Use / When Not

**Use when:**
- You are debugging N+1 queries, slow requests, or unexpected view/event behavior in local Laravel development.
- You want per-request SQL, timing, and memory without wiring up a full profiler.
- You are on Laravel Octane and need a debug bar that survives persistent workers (4.x).

**Avoid when:**
- The environment is anything public-facing or shared — it leaks request internals by design.
- You need production performance monitoring; reach for an APM (Telescope for Laravel-native insight, or Clockwork/Sentry/Datadog) instead.
- You are chasing a performance problem that the instrumentation itself distorts — profile with Xdebug/Blackfire/SPX for accurate numbers.

## Alternatives

- laravel/telescope — official Laravel debug assistant; a separate dashboard rather than an in-page bar, and usable (carefully) beyond local.
- itsgoingd/clockwork — browser-devtools-panel profiler for PHP; less intrusive than an injected bar, works across AJAX cleanly.
- php-debugbar/php-debugbar — the upstream, framework-agnostic library; use it directly when you are not on Laravel.
- spatie/laravel-ray — paid desktop debug app; use when you want a persistent external debug console instead of an in-request panel.
- Xdebug / Blackfire / SPX — use these when you need accurate profiling and call-graph data rather than request-scoped collectors.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-09 | Repository created; integration of PHP Debug Bar with Laravel 4[^3]. |
| 3.x | — | Long-lived series across Laravel 5–10; broad collector set. |
| rename | ~2023 | Canonical Composer package moved to `fruitcake/laravel-debugbar`[^2]. |
| 4.x | ~2023 | Out-of-the-box Laravel Octane support; `config/octane.php` flush entry removed[^1]. |

## References

[^1]: Laravel Debugbar README — installation, collectors, cautions, and Octane notes. https://github.com/fruitcake/laravel-debugbar
[^2]: Package rename note (`barryvdh/laravel-debugbar` → `fruitcake/laravel-debugbar`), Packagist. https://packagist.org/packages/fruitcake/laravel-debugbar
[^3]: PHP Debug Bar upstream library. https://github.com/php-debugbar/php-debugbar

## Tags

php, laravel, debugging, profiler, developer-tools, toolbar, debugbar, sql-profiling, octane, dev-dependency
