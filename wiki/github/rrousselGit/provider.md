# rrousselGit/provider

> A thin wrapper around Flutter's InheritedWidget that turns dependency injection and state exposure into a few lines instead of a boilerplate class.

[GitHub repo](https://github.com/rrousselGit/provider) ·
[pub.dev package](https://pub.dev/packages/provider) ·
[License: MIT](https://github.com/rrousselGit/provider/blob/master/packages/provider/LICENSE)

## Overview

`provider` is a Flutter state-management and dependency-injection package by Rémi
Rousselet, first published in 2018[^1]. It does not invent a new reactive model:
it wraps Flutter's own `InheritedWidget` — the framework primitive for pushing
values down the widget tree — and removes the boilerplate of writing a new
`InheritedWidget` subclass by hand each time. In exchange you get lazy creation,
automatic disposal, devtool visibility, and O(1) lookups via `context.watch` /
`context.read` / `context.select`. It is a Flutter Favorite package and, for
years, was the state-management approach the official Flutter docs teach first[^2].

The defining tension of this project is authorial: the same maintainer later
built Riverpod (2020) explicitly as "Provider, but better" — compile-safe, not
dependent on `BuildContext`, and free of the `ProviderNotFoundException` runtime
failure mode. As a result `provider` occupies an unusual position: extremely
widely deployed and stable, yet with its own author steering greenfield projects
toward a successor. Treat `provider` as mature, low-churn infrastructure rather
than the frontier of Flutter state management. Because it is a wrapper, its
mental model is inseparable from `InheritedWidget`: providers are widgets that
live in the tree, and a value is only reachable by descendants below it. Most
confusion new users hit — reading in the wrong `BuildContext`,
`ProviderNotFoundException`, mutating a `ChangeNotifier` during build — traces
back to widget-tree semantics, not to the package.

## Getting Started

```bash
flutter pub add provider
```

```dart
class Counter extends ChangeNotifier {
  int _count = 0;
  int get count => _count;
  void increment() {
    _count++;
    notifyListeners();      // triggers rebuild of watchers
  }
}

// Expose it above the widgets that need it (created lazily, disposed for you).
void main() => runApp(
      ChangeNotifierProvider(
        create: (_) => Counter(),
        child: const MyApp(),
      ),
    );

// Read it. watch() rebuilds on change; read() (in callbacks) does not.
Widget build(BuildContext context) => Text('${context.watch<Counter>().count}');
onPressed: () => context.read<Counter>().increment();
```

## Architecture / How It Works

Every provider is a widget that installs an `InheritedWidget` (an
`InheritedProvider`) into the tree. `context.watch<T>()` walks up to the nearest
`InheritedProvider` exposing `T`, registers the calling element as a dependent,
and returns the value; this lookup is O(1) because Flutter maintains a type-keyed
map of inherited widgets per element, so it does not traverse the tree[^1]. When
the exposed object changes, only registered dependents rebuild.

The three access methods differ only in subscription behavior. `watch<T>()`
subscribes and may only be called during `build`. `read<T>()` returns the value
without subscribing and must be called outside `build` (in callbacks/lifecycle).
`select<T, R>(cb)` subscribes to a derived slice and rebuilds only when that
slice changes — the primary escape hatch for over-rebuilding. `Provider.of<T>(
context, listen: false)` is the older equivalent of `read`.

Provider variants layer on top of the base `Provider`:

- **`ChangeNotifierProvider`** — the common case; listens to a `ChangeNotifier`
  and calls `dispose()` when removed from the tree.
- **`FutureProvider` / `StreamProvider`** — expose the latest value of a future
  or stream, with a required `initialData` (a breaking change in 5.0).
- **`ProxyProvider` / `ProxyProviderN`** — combine values from N upstream
  providers into a derived object, re-running `update` when any dependency
  changes; `ChangeNotifierProxyProvider` funnels the result into a notifier.
- **`MultiProvider`** — pure sugar that flattens nested providers into a list;
  behavior is identical to hand-nesting.

The `create` vs `.value` constructor distinction is a genuine footgun the README
devotes real space to: `create` owns the object's lifecycle (creates lazily,
disposes it), while `.value` assumes the object is owned elsewhere and must be
used for pre-existing instances — using `.value` to create leaks disposal, and
using `create` on an external instance risks disposing a value still in use.

## Production Notes

- **`ProviderNotFoundException` is a runtime failure, not a compile error.** If a
  widget reads a type no ancestor exposes, the app throws at runtime. This is the
  single most-cited reason the author built Riverpod. Nullable reads
  (`context.watch<Model?>()`) turn the throw into a `null` when a provider may be
  absent by design.
- **Reading in the wrong context.** A frequent bug is `watch`/`read` in
  `initState` or from a `BuildContext` that is not a descendant of the provider.
  `read` in `initState` works only if you never need updates; otherwise read in
  `build`. Mutating a `ChangeNotifier` synchronously during build (e.g. firing an
  HTTP request in a build path) also throws — defer with a post-frame callback.
- **Rebuild scope.** `context.watch<T>()` rebuilds the whole enclosing `build`
  method on any change to `T`. For large models, reach for `select` or a
  `Consumer`/`Selector` subtree to narrow what rebuilds — the main perf lever.
- **One provider per type per subtree.** Lookup is by type `T`; two providers of
  the same type in one path means descendants see only the nearest. Distinct
  models need distinct types.

## When to Use / When Not

**Use when:**
- You want the lowest-friction, framework-idiomatic way to expose a
  `ChangeNotifier` or plain value down the tree.
- The project already uses `provider`, or you follow the official Flutter
  state-management tutorial, which is built around it.
- You want minimal dependencies and a mental model that stays close to
  `InheritedWidget`.

**Avoid when:**
- You're starting a greenfield app and want compile-time safety and
  `BuildContext`-free access — the author's own recommendation there is Riverpod.
- Your state is event-driven or benefits from strict, testable separation of
  events and state — Bloc fits better.
- You need many providers of the same type, or global access without a widget
  ancestor — provider's tree-scoped, type-keyed model fights you.

## Alternatives

- rrousselGit/riverpod — the same author's successor; use it for new projects
  wanting compile-safe, context-independent providers without runtime "not found"
  exceptions.
- felangel/bloc — use when you want an explicit event → state pipeline with
  strong testability and clear architectural boundaries.
- mobxjs/mobx.dart — use when you prefer observable/reactive programming with
  automatic dependency tracking instead of manual `notifyListeners`.
- jonataslaw/getx — use when you want an all-in-one bundle (state + routing + DI)
  and accept its opinionated, non-idiomatic conventions.
- flutter/flutter — use raw `InheritedWidget`/`InheritedNotifier` directly when
  you want zero third-party dependencies and are willing to write the boilerplate
  `provider` removes.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2018 | Initial release: `InheritedWidget` wrapper[^1]. |
| 3.0.0 | 2019 | `ProxyProvider` family introduced[^3]. |
| 4.1.0 | 2020 | `context.watch` / `read` / `select` extension methods[^3]. |
| 5.0.0 | 2021 | Null-safety migration; `initialData` required for Future/Stream providers; `ValueListenableProvider` removed[^3]. |
| 6.0.0 | 2021 | Cleanup release; removed long-deprecated APIs[^3]. |

## References

[^1]: `provider` package README and API docs. https://pub.dev/packages/provider
[^2]: Flutter Favorite program listing and official simple state-management
  guide. https://docs.flutter.dev/data-and-backend/state-mgmt/simple
[^3]: `provider` changelog on pub.dev. https://pub.dev/packages/provider/changelog

## Tags

dart, flutter, state-management, dependency-injection, inheritedwidget, changenotifier, mobile, reactive, provider
