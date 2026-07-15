# rrousselGit/riverpod

> A reactive caching and state-management framework for Dart and Flutter that moves dependency lookup from runtime to compile time.

[GitHub repo](https://github.com/rrousselGit/riverpod) ·
[Official website](https://riverpod.dev) ·
[License: MIT](https://github.com/rrousselGit/riverpod/blob/master/LICENSE)

## Overview

Riverpod is a state-management and dependency-injection framework for Dart, written by Remi Rousselet — the same author as the `provider` package it supersedes (and whose name it anagrams)[^1]. It began in 2020 as a rewrite of `provider` intended to fix that package's structural flaws: reliance on `BuildContext` for reads, runtime `ProviderNotFoundException` errors, and the inability to expose two values of the same type. As of 2026 it is a Flutter Favorite and one of the two dominant state-management choices in the Flutter ecosystem alongside `bloc`.

The defining idea is that providers are declared as top-level global variables but hold no state themselves — they are immutable identifiers. Actual state lives in a `ProviderContainer` (held by a `ProviderScope` widget in Flutter apps). Because a provider is a compile-time constant used as its own key, the compiler catches missing or mistyped dependencies, and reads no longer need a widget's `context`. This is the whole pitch and the whole tradeoff: you trade `provider`'s runtime simplicity for a larger API surface and a steeper learning curve.

Riverpod's second defining tension is that it offers two ways to declare almost everything: hand-written provider objects (`FutureProvider`, `NotifierProvider`, etc.) and annotation-based code generation (`@riverpod` + `build_runner`). The docs and community have shifted toward code generation as the recommended path, but both remain supported, and the duplication is a common source of confusion for newcomers[^2].

## Getting Started

```bash
flutter pub add flutter_riverpod riverpod_annotation
flutter pub add --dev riverpod_generator build_runner custom_lint riverpod_lint
```

```dart
// main.dart — ProviderScope must wrap the app; it owns the state container.
void main() => runApp(const ProviderScope(child: MyApp()));

// counter.dart — code-generation style
part 'counter.g.dart'; // run: dart run build_runner watch

@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;               // initial state
  void increment() => state++;    // mutate; listeners rebuild
}

// In a widget:
class Home extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);          // subscribe + rebuild
    return TextButton(
      onPressed: () => ref.read(counterProvider.notifier).increment(),
      child: Text('$count'),
    );
  }
}
```

## Architecture / How It Works

The core is a directed dependency graph. Each provider is a node; `ref.watch(other)` inside a provider or widget creates an edge. When a node's value changes, Riverpod recomputes every downstream node and rebuilds the widgets that watch them. Reads come in three flavors with distinct semantics: `ref.watch` (subscribe and react to changes — use in `build`), `ref.read` (one-shot read, no subscription — use in callbacks), and `ref.listen` (side-effect callback on change). Misusing these is the single most common Riverpod bug.

The `provider` package resolved dependencies by walking the widget tree and matching on runtime `Type`, which produced `ProviderNotFoundException` and prevented two providers of the same type. Riverpod replaces this with global provider objects as identity keys, so lookup is a graph traversal in a `ProviderContainer` rather than a tree walk, and mismatches surface at compile time. The `ProviderScope` `InheritedWidget` only exposes the container; it is not itself the dependency mechanism.

Two modifiers shape lifecycle. `.autoDispose` destroys a provider's state once nothing listens to it (important for screen-scoped data); `.family` parameterizes a provider by an argument, effectively a memoized map of provider instances keyed by that argument. Code generation applies both automatically based on function signatures, which is why the generated and hand-written APIs diverge in behavior, not just syntax. The framework splits across packages: `riverpod` (pure Dart, no Flutter), `flutter_riverpod` (widgets: `ConsumerWidget`, `ProviderScope`), `hooks_riverpod` (integration with `flutter_hooks`), and the generator/lint toolchain.

## Production Notes

**`ref.read` vs `ref.watch` is a recurring footgun.** Watching in a callback creates stale closures; reading in `build` skips updates. `riverpod_lint` (via `custom_lint`) flags many of these statically and is effectively mandatory on serious projects — but `custom_lint` runs as a separate analyzer process and is a known source of "lints not showing up in IDE" friction.

**`.autoDispose` disposes state when you navigate away.** Data you expected to survive a back-navigation silently resets. `ref.keepAlive()` or a `KeepAliveLink` with a timer is the escape hatch; forgetting it, or over-applying `keepAlive`, causes either lost state or memory that never frees. `.family` + `.autoDispose` interactions are the subtle version of this.

**Code generation adds a build step.** Large codebases pay real `build_runner` time; `dart run build_runner watch` is the standard workaround, but generated `.g.dart` files and merge conflicts are part of daily life. Teams that avoid codegen keep more boilerplate but no build step.

**Global providers complicate tests only slightly.** Because providers are global, tests override them via `ProviderContainer(overrides: [...])` or `ProviderScope(overrides: [...])` rather than constructor injection. This is clean once learned but surprises developers expecting classic DI.

**Migration churn is real.** `StateProvider`, `StateNotifierProvider`, and `ChangeNotifierProvider` are legacy; the current model is `Notifier`/`AsyncNotifier` (2.0) plus code generation. Following older tutorials leads to deprecated APIs. The 2.x → 3.x transition consolidated the sync/async class hierarchy and adjusted error propagation semantics, so pinning a major version and reading its migration guide before upgrading is advised[^3].

## When to Use / When Not

**Use when:**
- You want compile-time-safe dependency wiring and testable, `context`-free business logic in Flutter.
- Your app has non-trivial async data (caching, loading/error states, pull-to-refresh) — `AsyncValue` handles these well.
- You want a single tool spanning DI and reactive state without a heavy event/bloc ceremony.

**Avoid when:**
- The app is small and `setState` or plain `provider` already suffices — Riverpod's API surface is overhead you don't need.
- Your team prefers an explicit, event-driven, highly structured architecture — `bloc` enforces more discipline by design.
- You want to avoid a code-generation build step and the two-ways-to-do-everything ambiguity.

## Alternatives

- rrousselGit/provider — the predecessor; simpler and lighter, use when you don't need compile-time safety or async caching.
- felangel/bloc — event/state driven with more ceremony; use when you want enforced, auditable state transitions.
- rodydavis/signals.dart — fine-grained reactive signals; use when you want minimal boilerplate and signal-style reactivity.
- mobxjs/mobx.dart — observable/reaction model with codegen; use when you prefer transparent reactive mutation.
- jonataslaw/getx — all-in-one state/routing/DI; use when you want batteries-included at the cost of architectural discipline.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-04 | Repository created; rewrite of `provider` addressing context-free, compile-safe reads[^1]. |
| 1.0 | 2021-11 | First stable release; `ProviderScope`, `ConsumerWidget`, family/autoDispose modifiers[^4]. |
| 2.0 | 2022-10 | `Notifier`/`AsyncNotifier`, `@riverpod` code generation, `riverpod_lint`; `StateNotifier`/`StateProvider` become legacy[^2]. |
| 3.0 | 2025 | Unified sync/async class hierarchy, revised error/retry handling; migration guide required[^3]. |

## References

[^1]: Riverpod README and repository — "Welcome to Riverpod (anagram of Provider)." https://github.com/rrousselGit/riverpod
[^2]: Riverpod docs, "About code generation." https://riverpod.dev/docs/concepts/about_code_generation
[^3]: Riverpod docs, "Migration / What's new in 3.0." https://riverpod.dev/docs/whats_new
[^4]: pub.dev package page for `riverpod`, release history. https://pub.dev/packages/riverpod

## Tags

dart, flutter, state-management, dependency-injection, reactive, caching, code-generation, async, mobile
