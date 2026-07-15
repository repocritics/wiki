# flutter-it/get_it

> A minimal service locator for Dart and Flutter: register objects once, resolve them by type from anywhere — no BuildContext, no code generation.

[GitHub repo](https://github.com/flutter-it/get_it) ·
[pub.dev package](https://pub.dev/packages/get_it) ·
[License: MIT](https://github.com/flutter-it/get_it/blob/master/LICENSE)

## Overview

get_it is a service locator for Dart, maintained by Thomas Burkhart (`@escamoteur`)[^1]. It solves one narrow problem: giving code access to shared objects (services, repositories, view models) without threading them through constructors or Flutter's widget tree. You register an object against a type at startup, then call `getIt<T>()` to retrieve it from anywhere — business logic, isolate-free background code, or a widget with no `BuildContext` in reach.

It is deliberately a *service locator*, not a dependency-injection framework, and that distinction is the whole design tension. In Martin Fowler's taxonomy[^2] a locator hides a class's dependencies: they no longer appear in the constructor signature, so you cannot tell what an object needs by reading its API. Critics call this an anti-pattern; get_it's stance is that the ergonomic win — zero boilerplate, no `build_runner`, O(1) map lookups, works in pure Dart — is worth the tradeoff, and that you can still use constructor injection *through* the locator when you want dependencies explicit (`UserRepo(getIt<ApiClient>())`).

The repository moved from the `fluttercommunity` org to `flutter-it` and is now positioned as the base of a small ecosystem (watch_it for state, command_it for commands, listen_it for reactive value operators)[^1]. get_it itself has no dependency on Flutter and runs in CLI tools and server-side Dart. Note that GitHub reports the repo language as JavaScript; that reflects the bundled documentation site, not the package, which is Dart.

## Getting Started

```yaml
# pubspec.yaml
dependencies:
  get_it: ^8.0.2
```

```dart
import 'package:get_it/get_it.dart';

final getIt = GetIt.instance; // process-global singleton container

class ApiClient {}
class UserRepository {
  UserRepository(this.api);
  final ApiClient api;
}

void configureDependencies() {
  // eager: instance created now
  getIt.registerSingleton<ApiClient>(ApiClient());
  // lazy: factory runs on first resolve, then cached
  getIt.registerLazySingleton<UserRepository>(
    () => UserRepository(getIt<ApiClient>()),
  );
  // factory: new instance on every resolve
  getIt.registerFactory<UserRepository>(() => UserRepository(getIt()));
}

// anywhere in the app
final repo = getIt<UserRepository>();
```

## Architecture / How It Works

Internally get_it is a `Map` keyed by `Type` (and optionally an `instanceName` string) pointing at a registration record that stores the factory closure, the instance once created, its registration mode, and any async/disposal state. Resolution is a hash-map lookup — there is no reflection, no scanning, and no generated code. `registerFactoryParam` allows passing up to two runtime arguments into the factory at resolve time.

Registration modes map to object lifetimes: `registerSingleton` (created eagerly at registration), `registerLazySingleton` (created on first access, then cached), and `registerFactory` (created fresh every call). Because keys are types, registering an interface against a concrete implementation (`registerSingleton<Auth>(FirebaseAuth())`) is the normal way to decouple call sites from implementations.

**Scopes** are a stack. `pushNewScope()` layers a new registration frame on top; a resolve walks from the top scope down, so a registration in a higher scope *shadows* a lower one of the same type. Popping the scope disposes its objects and restores what was shadowed. This models session/login state cleanly: push a scope at login with user-specific services, pop it at logout.

**Async initialization** is handled by `registerSingletonAsync` and `registerSingletonWithDependencies`. get_it can order async singletons by declared dependencies and expose `allReady()` / `isReady<T>()` futures that complete once initialization finishes, plus a manual `signalReady` path. This is the mechanism for orchestrating startup (open a database, hydrate config, connect a socket) before the first frame.

The container is mutable global state by construction. `GetIt.instance` is a process-wide singleton; you can also instantiate independent `GetIt` containers via `GetIt.asNewInstance()` if you need isolation.

## Production Notes

- **Runtime failures, not compile-time.** A missing or misordered registration throws at resolve time, not at build. There is no compile-time guarantee that `getIt<T>()` will succeed — this is the core safety gap versus code-gen (injectable) or compile-safe (riverpod) approaches. Centralize registration in one `configureDependencies()` and call it before `runApp`.
- **Registration order matters for eager singletons.** `registerSingleton` runs its constructor immediately, so a singleton that resolves another singleton in its constructor must be registered *after* its dependency, or use `registerLazySingleton` / `registerSingletonWithDependencies` to defer.
- **Testing requires teardown.** Because the container is global, state leaks between tests. Call `await getIt.reset()` in `tearDown`. Use `allowReassignment = true` or `reset()` to swap real services for mocks; forgetting this produces "already registered" assertions or cross-test contamination.
- **Type erasure caveats.** Registration is keyed on the static type argument. Registering with an inferred/dynamic type, or resolving with a different type than you registered, silently misses. Always specify the type parameter explicitly on `register…<T>()`.
- **Hidden dependencies.** The locator hides what a class needs. In large codebases this hurts readability and refactoring; teams often adopt a convention of resolving all dependencies in the constructor (not scattered through methods) to keep the coupling surface visible.
- **Disposal.** Singletons and scopes can register `dispose` callbacks, but only fire them on `reset`/scope-pop/`unregister` — a long-lived `GetIt.instance` never disposes on its own. Async disposal is supported and should be `await`ed.

## When to Use / When Not

**Use when:**
- You need shared services reachable from non-widget code (pure Dart, CLI, isolates, background logic).
- You want DI ergonomics without `build_runner` / annotations / a codegen step.
- You want to swap implementations behind an interface with minimal ceremony.
- You need lightweight lifecycle/scoping (login/logout, feature sessions) and async startup ordering.

**Avoid when:**
- You want compile-time guarantees that every dependency is wired — prefer riverpod or a codegen container.
- You want reactive UI that rebuilds when data changes — get_it alone does not; pair it with watch_it or use riverpod/provider.
- Your team considers service locators an anti-pattern and wants dependencies visible in every constructor signature.
- You need per-widget-subtree scoping tied to the element tree — that is provider/InheritedWidget territory.

## Alternatives

- rrousselGit/riverpod — use when you want compile-safe, reactive dependency graphs without a global mutable locator.
- rrousselGit/provider — use when injection should be scoped to the widget tree via `BuildContext` and `InheritedWidget`.
- milad-akarie/injectable — use when you want annotation-driven, compile-time wiring; it generates the registrations and runs on top of get_it.
- jonataslaw/getx — use when you want an all-in-one (routing + state + locator) and accept a large, opinionated surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018 | Initial release: singleton and factory registration[^3]. |
| 4.0 | 2020 | Scopes (push/pop, shadowing) introduced. |
| 7.0 | 2021 | API cleanup; async registration and readiness refinements. |
| 8.0 | 2024 | Current major line; latest published `8.0.2`[^4]. |

## References

[^1]: get_it README and flutter_it ecosystem (watch_it, command_it, listen_it), maintainer `@escamoteur`. https://github.com/flutter-it/get_it
[^2]: Martin Fowler, "Inversion of Control Containers and the Dependency Injection pattern" — service locator vs. dependency injection. https://martinfowler.com/articles/injection.html
[^3]: Repository created 2018-05-20. https://github.com/flutter-it/get_it
[^4]: get_it on pub.dev — versions and changelog. https://pub.dev/packages/get_it

## Tags

dart, flutter, service-locator, dependency-injection, ioc, state-management, di-container, no-codegen, singleton
