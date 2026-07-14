# laravel/laravel

> The application starter skeleton for Laravel — the repo you scaffold from, not the framework code itself.

[GitHub repo](https://github.com/laravel/laravel) ·
[Official website](https://laravel.com) ·
[License: MIT](https://github.com/laravel/laravel/blob/13.x/LICENSE)

## Overview

`laravel/laravel` is the starter template for a new Laravel application — the directory
layout, config files, `composer.json`, and a handful of stub classes you get when you run
`composer create-project laravel/laravel` or `laravel new`. The actual framework lives in a
separate repository, **laravel/framework**, which is assembled from the `illuminate/*`
component packages and pulled in as a Composer dependency[^1]. Editing a bug in Eloquent or
the router means opening laravel/framework, not this repo. Confusing the two is the single
most common orientation mistake for newcomers, and it matters for this catalog: the 84k stars
here largely reflect the framework's popularity, not the skeleton's code.

Laravel itself is a full-stack PHP web framework created by Taylor Otwell, first released in
2011[^2]. Since Laravel 4 (2013) it has been Composer-based and built from the decoupled
Illuminate components. It is the dominant PHP application framework: Eloquent ORM (Active
Record), Blade templating, the Artisan CLI, queues, broadcasting, and a first-party product
line (Forge, Vapor, Horizon, Sail, Herd, Nova, Reverb) sit around a service-container core.
Its defining tradeoff is convenience versus magic: facades, auto-resolving dependency
injection, and Eloquent's Active Record make common tasks terse, but the same indirection
hides query cost, lifecycle, and state — problems that only surface under production load.

## Getting Started

```bash
# Scaffold a new app from this skeleton
composer create-project laravel/laravel my-app
cd my-app
php artisan serve
```

```php
// routes/web.php — closure route
use Illuminate\Support\Facades\Route;

Route::get('/posts/{post}', function (App\Models\Post $post) {
    return view('posts.show', ['post' => $post]);   // route-model binding
});
```

```php
// app/Models/Post.php — an Eloquent model
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    protected $fillable = ['title', 'body'];   // mass-assignment allow-list

    public function author()
    {
        return $this->belongsTo(User::class);
    }
}
```

## Architecture / How It Works

The framework is a **monorepo split**: laravel/framework is developed as one repo, then
`git subtree`-split into read-only `illuminate/database`, `illuminate/routing`,
`illuminate/queue`, etc., so apps can require individual components[^1]. This skeleton pins
`laravel/framework` in `composer.json` and wires it together through config and providers.

Core mechanisms:

- **Service container** — a dependency-injection container resolves classes and their
  constructor dependencies automatically. Service providers (registered in
  `bootstrap/providers.php`) bind services during boot.
- **Facades** — static-looking accessors (`Cache::get()`, `DB::table()`) that proxy to
  container-resolved singletons. Convenient; they obscure the real dependency and complicate
  static analysis and mocking.
- **Eloquent** — an Active Record ORM: models are both query builders and row objects.
  Relationships are lazy-loaded by default, which is where N+1 queries come from.
- **Blade** — templates compiled to plain PHP and cached; directives (`@if`, `@foreach`,
  components) are the surface.
- **Artisan** — the CLI, built on Symfony Console. Laravel also depends on Symfony's
  HttpFoundation, Console, and other components under the hood[^3].

Since **Laravel 11 (2024)** the skeleton was slimmed dramatically: middleware, exception
handling, and service-provider registration moved out of stub files and into
`bootstrap/app.php` and framework defaults, so a fresh app ships far fewer files than pre-11
versions[^4]. This is the biggest structural change most existing tutorials predate.

## Production Notes

**Eloquent N+1 queries.** The default lazy loading means iterating a collection and touching
a relation issues one query per row. Eager-load with `->with('relation')`. In development,
`Model::preventLazyLoading()` throws on any lazy load so you catch them before production.

**Caching is mandatory for deploys.** `php artisan config:cache`, `route:cache`,
`view:cache`, and `event:cache` collapse config/route resolution into compiled files. Skipping
them leaves measurable per-request overhead; forgetting to re-run them after a config change
leads to "why is my change not taking effect" confusion. Pair with PHP OPcache.

**Queues need real workers.** `queue:work` (not `queue:listen`) run under Supervisor, or
Horizon for Redis-backed queues with dashboards and autoscaling. Long-lived workers hold a
booted framework in memory, so deploys must `queue:restart` or stale code keeps running.

**Octane state leakage.** Laravel Octane (FrankenPHP, Swoole, or RoadRunner) keeps the app
booted across requests for large throughput gains, but breaks the traditional
"fresh state per request" assumption. Singletons, static properties, and request data held on
long-lived services leak between users unless reset — a class of bug that does not exist in
standard PHP-FPM deployment[^5].

**Upgrade cadence.** Laravel ships a major version roughly yearly (now targeting Q1); each
release gets ~18 months of bug fixes and ~2 years of security fixes[^6]. Upgrades are usually
mechanical via the official upgrade guide, but Symfony-component bumps and PHP minimum-version
raises can cascade into third-party packages. Laravel Shift (paid) automates most of it.

**Mass assignment.** `$fillable` / `$guarded` on models gate which attributes accept bulk
input. A permissive `$guarded = []` plus `Model::create($request->all())` is a real
privilege-escalation footgun.

## When to Use / When Not

**Use when:**
- You're building a database-backed web application or API in PHP and want batteries included.
- You value a large ecosystem, hosting story (Forge/Vapor/Cloud), and hire-able talent pool.
- Your team benefits from strong conventions and first-party auth, queues, and broadcasting.

**Avoid when:**
- You want an explicit, magic-free architecture — Symfony or a micro-framework fit better.
- You're writing a small library or CLI where a full framework is dead weight (use Slim).
- Your workload is not request/response web (heavy data pipelines, ML) — PHP is the wrong tool.
- You need minimal, predictable runtime state and can't budget for Octane's leakage caveats.

## Alternatives

- symfony/symfony — decoupled, component-first, more explicit and enterprise-oriented; use it when you want less magic and Laravel already borrows its components.
- slimphp/Slim — micro-framework; use when you want routing + PSR-7 and nothing else.
- cakephp/cakephp — convention-over-configuration full-stack alternative with its own ORM.
- yiisoft/yii2 — mature full-stack PHP framework; use for teams already invested in it.
- rails/rails — the Ruby framework Laravel is philosophically closest to; use when the team is Ruby, not PHP.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011-06 | Initial release by Taylor Otwell[^2]. |
| 4.0 | 2013-05 | Composer-based rewrite on Illuminate components[^1]. |
| 5.0 | 2015-02 | New directory structure, Elixir, method injection. |
| 6.0 | 2019-09 | Semantic versioning adopted; LTS. |
| 8.0 | 2020-09 | Jetstream, model factory classes, Sail. |
| 9.0 | 2022-02 | Symfony 6, PHP 8 minimum. |
| 11.0 | 2024-03 | Slimmed application skeleton; `bootstrap/app.php`[^4]. |
| 12.0 | 2025-02 | Yearly Q1 cadence continues; starter-kit refresh. |
| 13.x | 2026 | Current release line (default branch)[^7]. |

## References

[^1]: Laravel framework repository and Illuminate component split. https://github.com/laravel/framework
[^2]: Laravel history / Taylor Otwell first release, 2011. https://en.wikipedia.org/wiki/Laravel
[^3]: Laravel uses Symfony components (Console, HttpFoundation, etc.). https://laravel.com/docs/releases
[^4]: Laravel 11 slimmed skeleton and `bootstrap/app.php`. https://laravel.com/docs/11.x/upgrade
[^5]: Laravel Octane state / concurrency caveats. https://laravel.com/docs/octane
[^6]: Laravel support policy (bug/security windows). https://laravel.com/docs/releases#support-policy
[^7]: Repository default branch `13.x` (fetched via GitHub API, 2026-07). https://github.com/laravel/laravel

## Tags

php, laravel, web-framework, full-stack, eloquent-orm, mvc, blade, artisan, backend, starter-template
