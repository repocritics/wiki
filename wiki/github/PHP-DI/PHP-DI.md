# PHP-DI/PHP-DI

> A PSR-11 dependency injection container for PHP built around autowiring, with an optional compilation step for production.

[GitHub repo](https://github.com/PHP-DI/PHP-DI) ·
[Official website](https://php-di.org) ·
[License: MIT](https://github.com/PHP-DI/PHP-DI/blob/master/LICENSE)

## Overview

PHP-DI is a dependency injection container created by Matthieu Napoli (`mnapoli`) in 2012[^1]. Its stated goal is to be practical and framework-agnostic: you can drop it into a plain PHP project, or bridge it into Symfony, Slim, or a legacy codebase, and let it wire object graphs together from constructor type hints. It implements the PSR-11 `ContainerInterface`[^2], the standard `get()`/`has()` contract that most modern PHP containers now share — and its maintainer was a driver of `container-interop`, the predecessor effort that became PSR-11.

The defining feature is autowiring: given a class whose constructor type-hints its dependencies, PHP-DI resolves the whole tree by reflection with no configuration. The defining tension is what that convenience costs. Reflection at runtime is slow, so anything performance-sensitive is expected to run the container through a compilation step that generates a plain PHP class; and autowiring only reaches as far as type hints do — scalar arguments, interface-to-implementation bindings, and factory logic all still require explicit definitions. PHP-DI therefore sits between the "zero-config magic" containers (Laravel's) and the "everything is compiled config" containers (Symfony's), and inherits obligations from both sides.

## Getting Started

```bash
composer require php-di/php-di
```

```php
use DI\ContainerBuilder;
use function DI\create;
use function DI\get;

$builder = new ContainerBuilder();
$builder->addDefinitions([
    // interface -> concrete, with an explicit constructor arg
    LoggerInterface::class => create(FileLogger::class)
        ->constructor(get('log.file')),
    'log.file' => '/var/log/app.log',
]);
$container = $builder->build();

// FileLogger is constructed and injected wherever LoggerInterface is type-hinted
$service = $container->get(UserService::class); // autowired, no definition needed
```

Any class with type-hinted constructor arguments (`UserService` above) is resolved without a definition. Definitions are only written for the parts autowiring cannot infer: interfaces, scalars, and factories.

## Architecture / How It Works

PHP-DI is built as a set of definition *sources* resolved on demand and cached in memory for the lifetime of the container (services are singletons unless declared otherwise).

- **Autowiring** reads constructor signatures via PHP reflection. For a type-hinted parameter it recursively resolves that type; untyped or scalar parameters raise an error unless a definition supplies them.
- **Definitions** are PHP arrays keyed by entry name, using helper functions: `create()` (instantiate a class), `autowire()` (autowire but override some args), `get()` (reference another entry), `factory()` (a closure/callable that builds the value), `value()`, `env()`, and `decorate()`. Definitions can be split across multiple files and merged.
- **Attributes** (`#[Inject]`) opt into property and method injection. As of PHP-DI 7 these are native PHP 8 attributes; earlier versions used Doctrine-style docblock annotations, which were removed in the 7.0 rewrite[^3]. Attribute injection is off by default and must be enabled on the builder.
- **Compilation.** `ContainerBuilder::enableCompilation()` writes a generated container class to disk so that, on subsequent requests, resolution is direct method calls instead of reflection[^4]. This was introduced in PHP-DI 6 and is the intended production configuration.
- **Lazy injection** wraps a dependency in a proxy that defers construction until first use, via a proxy-manager library. This adds a dependency and a code-generation step, so it is opt-in per entry.

The container itself is small; most of the surface area is the definition language and the reflection/compilation machinery behind it.

## Production Notes

**Compile in production, or pay reflection on every request.** Uncompiled, PHP-DI reflects on classes at runtime — fine for development, measurably slower under load. `enableCompilation(__DIR__ . '/var/cache')` generates the container once. The cache directory must be writable at deploy time and is not auto-invalidated: changing definitions requires deleting the compiled file, so deployment scripts must clear it (or write to a release-specific path).

**Not every closure compiles.** Factory definitions written as closures are only compilable when they are "plain" — a closure that binds `$this`, captures variables via `use`, or is otherwise stateful cannot be serialized into the compiled container and will throw at compile time. The practical fix is to move such logic into an invokable class referenced by `factory()`. This is the most common surprise when a codebase first turns compilation on.

**Autowiring stops at the type system.** Interfaces must be mapped to implementations explicitly; scalar/config values must be defined; union and intersection types, variadic constructors, and untyped parameters need help. Teams that lean hard on autowiring can end up with a container that "mostly configures itself" but fails opaquely at runtime for the entries it could not infer — errors surface at `get()` time, not at build time (unlike Symfony's compile-time validation).

**Lazy injection has costs.** It pulls in a proxy-manager dependency and writes proxy classes to disk (`writeProxiesToFile()`), which is another cache directory to manage. Use it only for genuinely expensive-to-construct dependencies.

**Framework integration is a bridge, not a merge.** The Symfony bridge and Slim/PSR-11 integrations let PHP-DI back another framework's container, but you then run two definition styles side by side and must understand which container owns which service. This is workable but is a source of confusion in mixed codebases.

## When to Use / When Not

**Use when:**
- You want autowiring in a framework-agnostic form — a standalone app, a Slim project, or a legacy codebase without a DI container.
- You prefer expressive PHP-array definitions over XML/YAML.
- You want PSR-11 compliance so the container is swappable.
- You're comfortable running a compile step in your deploy pipeline.

**Avoid when:**
- You're already all-in on Symfony or Laravel — their native containers are compile-time-validated (Symfony) or deeply integrated (Laravel), and adding PHP-DI duplicates concerns.
- You want configuration errors caught at build time rather than at first `get()`.
- You need the absolute minimum footprint — a lightweight PSR-11 container without reflection/compilation machinery is leaner.
- Your object graph is trivial enough that a hand-written factory or a service-locator like Pimple is simpler.

## Alternatives

- symfony/dependency-injection — the reference PHP container; compile-time validation and vast ecosystem, but heavier and most natural inside Symfony. Use when you want errors caught at build time or you're already in the Symfony world.
- laravel/framework (illuminate/container) — autowiring container tightly bound to Laravel's lifecycle. Use when you're building on Laravel and want its facades/bindings.
- league/container — small, fast, PSR-11 container with explicit service providers. Use when you want autowiring-optional minimalism over PHP-DI's reflection-heavy defaults.
- silexphp/Pimple — minimal closure-based service container (really a service locator). Use when the graph is small and you want almost no abstraction.
- laminas/laminas-servicemanager — factory-driven container from the Laminas stack. Use when you're in a Laminas/Mezzio application.

## History

| Version | Date | Notes |
|---------|------|-------|
| 4.0 | 2014-02-04 | Definition array config, early autowiring model[^5]. |
| 5.0 | 2015-06-10 | `container-interop` support (pre-PSR-11). |
| 6.0 | 2018-02-20 | PSR-11 `ContainerInterface`, container compilation, PHP 7 requirement[^4]. |
| 6.3 | 2020-10-12 | PHP 8 support. |
| 6.4 | 2022-04-09 | Final 6.x line. |
| 7.0 | 2023-01-12 | PHP 8 required; native attributes replace Doctrine annotations[^3]. |
| 7.1.1 | 2025-08-16 | Latest release at time of writing. |

## References

[^1]: PHP-DI repository, created 2012-03-17. https://github.com/PHP-DI/PHP-DI
[^2]: PSR-11 Container Interface, PHP-FIG. https://www.php-fig.org/psr/psr-11/
[^3]: PHP-DI 7 migration guide (attributes replace annotations). https://php-di.org/doc/migration/7.0.html
[^4]: PHP-DI documentation, "Compiling the container" and PHP-DI 6 introduction. https://php-di.org/doc/performances.html
[^5]: PHP-DI documentation and change log. https://php-di.org/doc/

## Tags

php, dependency-injection, ioc-container, psr-11, autowiring, framework-agnostic, container, service-container, reflection, mit-license
