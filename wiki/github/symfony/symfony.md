# symfony/symfony

> A PHP framework built as a set of decoupled, reusable components — and the development monorepo those components are split out of.

[GitHub repo](https://github.com/symfony/symfony) ·
[Official website](https://symfony.com) ·
[License: MIT](https://github.com/symfony/symfony/blob/8.2/LICENSE)

## Overview

Symfony is a full-stack PHP web framework and, underneath it, a catalog of
standalone components (HttpFoundation, Routing, Console, DependencyInjection,
EventDispatcher, Form, Validator, Serializer, Messenger, Cache, and dozens
more). It was created by Fabien Potencier at SensioLabs; the modern
component-based architecture arrived with the ground-up Symfony 2.0 rewrite in
2011[^1]. The framework you install is a thin assembly layer (FrameworkBundle)
over those components, so most of Symfony's surface area is usable without the
framework at all.

This is the project's defining trait: Symfony is infrastructure other projects
build on. Laravel, Drupal, phpBB, and Composer itself all depend on Symfony
components[^2]. That reach comes with a cultural bias toward explicitness,
configuration, and backward-compatibility guarantees rather than
convention-driven magic. The tradeoff is real — Symfony asks you to wire more
things up front and rewards long-lived, large applications, while feeling
heavier than micro-frameworks or Laravel for quick prototypes.

At ~31k stars and pushed to daily, `symfony/symfony` is among the most actively
maintained PHP projects in existence; the repository has run continuously since
2010 with a predictable, calendar-driven release train.

## Getting Started

```bash
# Composer (framework skeleton)
composer create-project symfony/skeleton my_app
cd my_app
composer require webapp   # optional: Twig, forms, asset handling, etc.

# Or the Symfony CLI (separate Go binary, includes a local web server)
symfony new my_app --webapp
symfony serve
```

```php
// src/Controller/HelloController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class HelloController extends AbstractController
{
    #[Route('/hello/{name}', name: 'hello')]
    public function index(string $name): Response
    {
        return new Response("Hello {$name}");
    }
}
```

Routing, controller resolution, and dependency injection are all driven by PHP
8 attributes and autowiring by default; the older annotation and heavy-YAML
styles still work but are no longer the recommended path.

## Architecture / How It Works

The core is **HttpKernel**: an HTTP request is turned into a response through a
fixed sequence of events — `kernel.request`, `kernel.controller`,
`kernel.view`, `kernel.response`, `kernel.exception`, and `kernel.terminate`[^3].
Almost every framework feature (routing, security, sessions, profiler) is a
listener on one of these events. Understanding this event flow is the single
most useful thing to know about Symfony internals.

The **DependencyInjection** container is compiled: service definitions from
attributes, YAML, or PHP are resolved once and dumped to a cached PHP class, so
runtime overhead is near zero. Autowiring resolves constructor type-hints to
services automatically. **Bundles** are the plugin unit — self-contained
packages that register services and configuration into the container.

`symfony/symfony` is a **monorepo**. The individual component packages you
`composer require` (e.g. `symfony/console`, `symfony/http-foundation`) are
read-only Git subtree splits published automatically from this repository[^4].
You develop and open pull requests against the monorepo; the split repos exist
so consumers can depend on one component without pulling the whole framework.

**Symfony Flex** is a Composer plugin that maps package names to "recipes" —
small manifests that scaffold config files, environment variables, and
directories when a package is installed[^5]. This is what makes
`composer require webapp` configure Twig and the asset pipeline for you.

## Production Notes

**Cache warmup is mandatory on deploy.** Symfony compiles the container,
routes, and template metadata into `var/cache/`. Run
`APP_ENV=prod bin/console cache:clear` (which warms the cache) as a build step;
serving a cold `prod` cache on the first request is slow and can fail on
read-only filesystems.

**OPcache and preloading matter.** Symfony ships a preload script
(`config/preload.php`) intended to be loaded via PHP's `opcache.preload`. For
serious throughput you want OPcache enabled with a generous memory limit and,
optionally, APCu for the metadata caches. Without OPcache, per-request autoload
and container bootstrap costs are noticeable.

**The upgrade path is deprecation-driven.** Symfony's backward-compatibility
promise means a minor release never breaks public API; instead it emits
runtime deprecation notices. The intended workflow is: stay on the latest minor
of your major, resolve every deprecation (the profiler and
`symfony/deprecation-contracts` surface them), then bump the major — which
mostly just removes the already-deprecated paths. Teams that skip several
majors at once lose this smoothness and face a large manual migration.

**Async needs worker processes.** The Messenger component dispatches to
transports (AMQP, Redis, Doctrine, etc.) but nothing is consumed until you run
`bin/console messenger:consume` under a process supervisor (systemd,
supervisord). Forgetting the worker is a common "my emails never send" bug.

**Mixed component versions cause conflicts.** Because components are usable
standalone, it is possible to end up with, say, `symfony/console` and
`symfony/http-kernel` on different minor versions; keep them aligned via a
version constraint or the `symfony/flex` `extra.symfony.require` pin.

## When to Use / When Not

**Use when:**
- You are building a long-lived, large application and value strict semantic
  versioning and a documented BC promise.
- You want reusable, framework-agnostic components (Console, Cache, Serializer)
  even outside a full app.
- You need mature Form, Security, and Validation subsystems, or you're building
  on API Platform.
- Your team prefers explicit configuration and type-safe wiring over convention
  magic.

**Avoid when:**
- You want a batteries-included, opinionated DX for shipping fast — Laravel
  covers that ground (and is itself built on Symfony components).
- You're writing a small script or micro-API where a full container and bundle
  system is overhead.
- Your team is unwilling to keep up with the ~6-month release cadence; falling
  many versions behind forfeits the smooth upgrade story.

## Alternatives

- laravel/laravel — batteries-included and convention-first; use it when you
  want speed of delivery and a rich first-party ecosystem (Eloquent, Nova,
  Forge). It depends on Symfony components under the hood.
- api-platform/api-platform — built on Symfony; use it when a REST/GraphQL API
  is the whole product rather than a full-stack app.
- slimphp/slim — a micro-framework; use it for small routers and APIs where a
  full DI container and bundle system are unnecessary.
- cakephp/cakephp — convention-over-configuration MVC; use it when you want an
  integrated ORM and scaffolding closer to Rails' style.
- laminas/laminas-mvc — the Zend Framework successor; use it in enterprises
  already standardized on that stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2007-01 | First stable release; MVC built on Mojavi/Propel-era ideas. |
| 2.0 | 2011-07 | Full rewrite: components, DI container, HttpKernel, Twig[^1]. |
| 3.0 | 2015-11 | API cleanup, removal of 2.x deprecations. |
| 4.0 | 2017-11 | Flex + recipes, minimal skeleton, `bin/console`[^5]. |
| 5.0 | 2019-11 | PHP 7.2+, Mailer/Notifier, HttpClient, Messenger maturing. |
| 6.0 | 2021-11 | PHP 8.0.2+ required, first-class attributes, Runtime component. |
| 7.0 | 2023-11 | PHP 8.2+ required; AssetMapper, Clock, Scheduler. |
| 8.0 | 2025-11 | Current major; released alongside 7.4 LTS[^6]. |

Symfony ships a new minor every ~6 months (May and November) and a new major
every two years in November, with the final minor of each major (2.8, 3.4, 4.4,
5.4, 6.4, 7.4) designated a Long-Term Support release[^6].

## References

[^1]: Symfony blog, "Symfony 2.0.0" release — 2011-07-28. https://symfony.com/blog/symfony-2-0-0
[^2]: "Symfony Components" — projects using Symfony. https://symfony.com/projects
[^3]: Symfony docs, "The HttpKernel Component: Events". https://symfony.com/doc/current/components/http_kernel.html
[^4]: Symfony docs, "Contributing Code" — the monorepo and subtree splits. https://symfony.com/doc/current/contributing/code/index.html
[^5]: Symfony docs, "Symfony Flex and recipes". https://symfony.com/doc/current/setup.html
[^6]: Symfony docs, "The Release Process" (semantic versioning, LTS). https://symfony.com/doc/current/contributing/community/releases.html

## Tags

php, web-framework, backend, mvc, components, dependency-injection, monorepo, http-kernel, symfony, composer, enterprise
