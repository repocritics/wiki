# thephpleague/container

> A small PSR-11 dependency injection container for PHP, built around explicit definitions, service providers, and inflectors rather than automatic wiring.

[GitHub repo](https://github.com/thephpleague/container) ·
[Official website](http://container.thephpleague.com) ·
[License: MIT](https://github.com/thephpleague/container/blob/master/LICENSE.md)

## Overview

Container is a dependency injection container maintained under The PHP League
umbrella. It began life as `Orno\Di` (by Phil Bennett) and was adopted into the
League, where it has been maintained as `league/container` since 2015[^1]. It is
compliant with PSR-11 (`Psr\Container\ContainerInterface`), so anything that
consumes a standard container interface can accept it.

Its defining characteristic is a preference for explicit registration over
magic. Out of the box, Container will not resolve an arbitrary class it has
never been told about. Autowiring (resolving constructor dependencies by
reflection) exists but is opt-in: you register a `ReflectionContainer` as a
delegate. This is the opposite default from Laravel's container or PHP-DI, and
it is the single most common point of surprise for newcomers who expect
`get(SomeClass::class)` to "just work"[^2].

The tradeoff is deliberate. Container aims to stay small and predictable — a
readable codebase, few moving parts, resolution behavior you can reason about —
at the cost of the ergonomics and compile-time optimization that larger
containers (Symfony DI, PHP-DI) provide. It targets library authors and
framework builders who want a compliant, unopinionated container, more than
application developers who want zero-config autowiring.

## Getting Started

```bash
composer require league/container
```

```php
use League\Container\Container;
use League\Container\ReflectionContainer;

$container = new Container();

// Opt into autowiring for classes not explicitly registered:
$container->delegate(new ReflectionContainer(true)); // true = cache resolutions

// Explicit definition:
$container->add(Logger::class);
$container->add(Mailer::class)
    ->addArgument(Logger::class);   // constructor arg by service id

// share() = one shared instance (singleton); add() = new instance per get()
$container->add(Config::class)->setShared(true);

$mailer = $container->get(Mailer::class);
```

## Architecture / How It Works

The container is a registry of **definitions**. `add()` and `share()` return a
`Definition` you configure fluently — `addArgument()` / `addArguments()` for
constructor parameters, `addMethodCall()` for setter injection, and a `tag()`
to group related services. Arguments are resolved lazily by service id when
`get()` is first called. `DefinitionAggregate` holds the collection and does the
lookup.

**Autowiring lives in `ReflectionContainer`**, a separate resolver registered
via `delegate()`. When the main container has no definition for an id, it walks
its delegate chain; `ReflectionContainer` inspects the constructor signature and
recursively resolves type-hinted parameters. Passing `true` to its constructor
caches resolved instances. Because this is a delegate rather than the default
path, a project can run fully explicit, fully autowired, or a mix.

**Service providers** are the packaging unit. A class extending
`AbstractServiceProvider` implements `provides(string $id): bool` and
`register()`; the container only boots a provider when something it advertises
is requested, which keeps registration lazy. `BootableServiceProviderInterface`
adds a `boot()` hook that runs eagerly when the provider is added.

**Inflectors** are the feature that most distinguishes Container from its peers.
`inflector(SomeInterface::class)` registers a manipulation applied to *every*
resolved object matching that type — e.g. call `setLogger()` on anything
implementing `LoggerAwareInterface`. This is cross-cutting setter injection
without touching each definition.

**Delegate containers** let Container defer to any other PSR-11 container,
which is how it composes with third-party containers and how autowiring is
bolted on. There is no compilation step: resolution happens at runtime on every
cold path, unlike Symfony DI, which dumps an optimized PHP container class.

## Production Notes

- **Autowiring is off until you delegate a `ReflectionContainer`.** If
  `get(App\Service::class)` throws `NotFoundException` for a class that plainly
  exists, this is almost always the cause. It is a design choice, not a bug.
- **No compiled container.** Resolution is reflection- and array-driven at
  runtime. For most apps this is irrelevant, but hot paths that resolve many
  autowired classes per request pay a reflection cost Symfony's dumped container
  avoids. Register frequently-used services explicitly, and enable the
  `ReflectionContainer` cache flag.
- **`add()` vs `share()` semantics bite people.** `add()` returns a fresh
  instance on every `get()`; `share()` (or `setShared(true)`) returns one shared
  instance. Getting this wrong produces either state leaks or unexpected
  duplication. This default (non-shared) is the reverse of some other containers.
- **Breaking changes across majors.** The definition/API surface has shifted
  between major versions (notably the 2.x → 3.x move to full PSR-11 compliance,
  and later PHP-version bumps). Upgrades are not always drop-in; read the
  changelog before bumping the constraint[^3].
- **Modern PHP only.** The current release line requires PHP 8.3–8.5[^4]. Legacy
  PHP 7 projects are stuck on older, unmaintained major versions.
- **No contextual bindings.** Unlike Laravel's container, there is no built-in
  "inject implementation A into consumer X but B into consumer Y" mechanism;
  model that with distinct service ids or provider logic.

## When to Use / When Not

**Use when:**
- You're building a framework or library and want a small, PSR-11-compliant
  container without dragging in a large dependency.
- You prefer explicit, auditable wiring over reflection magic.
- You want inflectors for cross-cutting setter injection (logger-aware, etc.).

**Avoid when:**
- You want zero-config autowiring as the default and contextual bindings —
  Laravel's container or PHP-DI fit better.
- You need compile-time container optimization for a large, hot codebase —
  Symfony DependencyInjection dumps an optimized container.
- You're on PHP 7 and cannot upgrade — the current line won't install.

## Alternatives

- php-di/php-di — use when you want autowiring-first resolution with attribute
  configuration and optional compilation.
- symfony/dependency-injection — use for large applications needing a compiled
  container, contextual bindings, and YAML/XML/PHP config.
- illuminate/container — use inside Laravel, or when you want zero-config
  autowiring with contextual bindings out of the box.
- silexphp/Pimple — use when you want the smallest possible closure-based
  container and don't need the full definition/provider machinery.
- rdlowrey/auryn — use when you want recursive autowiring with no configuration;
  note it is far less actively maintained.

## History

| Version | Date | Notes |
|---------|------|-------|
| Orno\Di | pre-2015 | Origin project by Phil Bennett, before League adoption[^1]. |
| 1.x | 2015 | First `league/container` releases; repo created 2015-01[^1]. |
| 2.x | ~2016 | Service providers, inflectors, delegate containers. |
| 3.x | ~2017 | Full PSR-11 (`ContainerInterface`) compliance[^2]. |
| 4.x | ~2020 | Dropped legacy PHP; modern typed API. |
| current | 2026 | PHP 8.3–8.5 support; last pushed 2026-03[^4]. |

## References

[^1]: The PHP League, "Container" project home. http://container.thephpleague.com
[^2]: PSR-11: Container Interface — PHP-FIG. https://www.php-fig.org/psr/psr-11/
[^3]: league/container releases and changelog. https://github.com/thephpleague/container/releases
[^4]: league/container README (requirements: PHP 8.3, 8.4, 8.5). https://github.com/thephpleague/container/blob/master/README.md

## Tags

php, dependency-injection, container, psr-11, ioc, service-provider, autowiring, the-php-league, composer-package
