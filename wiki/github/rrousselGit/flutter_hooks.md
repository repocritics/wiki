# rrousselGit/flutter_hooks

> React-style hooks for Flutter — reusable widget lifecycle logic without the `StatefulWidget` boilerplate.

[GitHub repo](https://github.com/rrousselGit/flutter_hooks) ·
[pub.dev package](https://pub.dev/packages/flutter_hooks) ·
[License: MIT](https://github.com/rrousselGit/flutter_hooks/blob/master/LICENSE)

## Overview

flutter_hooks is a direct port of React's hooks model to Flutter, written by Remi Rousselet — the same author behind Provider, Riverpod, and freezed[^1]. Its single purpose is code reuse: extracting the `initState` / `didUpdateWidget` / `dispose` lifecycle dance that every `StatefulWidget` re-implements by hand into composable functions like `useAnimationController()` and `useTextEditingController()`. A `HookWidget` builds like a stateless widget, but each `use*` call inside `build` transparently allocates and disposes state tied to that widget's `Element`.

The defining tension is that hooks are foreign to Flutter's design. Flutter's own team never adopted the model, so hooks live outside the framework's conventions: they rely on call *order* rather than named fields, they cannot be called conditionally or inside loops, and the compiler cannot enforce either rule. React solved this with a first-party ESLint plugin that ships with every project; the Flutter ecosystem has no equivalent blessed by the SDK, so the rules are enforced by discipline and, optionally, community lint packages. Adopting hooks means accepting a parallel idiom that most Flutter developers, tutorials, and Stack Overflow answers do not assume.

The payoff is real for widgets heavy in controllers and subscriptions — animation, forms, streams — where hooks collapse dozens of lines of ceremony into single calls. The package is mature and stable (first released 2018, null-safe since 2021) rather than fast-moving; the low commit velocity reflects a settled API, not abandonment[^2].

## Getting Started

```yaml
# pubspec.yaml
dependencies:
  flutter_hooks: ^0.21.0
```

```dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

class Counter extends HookWidget {
  const Counter({super.key});

  @override
  Widget build(BuildContext context) {
    // useState replaces a StatefulWidget + setState.
    final count = useState(0);
    // useAnimationController is created once and disposed automatically.
    final controller = useAnimationController(
      duration: const Duration(milliseconds: 300),
    );

    return TextButton(
      onPressed: () => count.value++,
      child: Text('${count.value}'),
    );
  }
}
```

Any widget mixing in `Hooks` works; `HookWidget` and `HookBuilder` are the common entry points.

## Architecture / How It Works

Instead of a single `State` object, a `HookElement` holds a `List<HookState>` plus an integer cursor. Each `use*()` call reads the hook at the current cursor position and increments it; on every rebuild the cursor resets to zero and the sequence replays[^3]. This is why hooks must be called unconditionally and in a stable order — the identity of a hook *is* its index. Wrap a `useState` in an `if` and on the next build every subsequent hook shifts by one slot and silently binds to the wrong state.

A hook is either a plain `use*` function composed from other hooks, or a class extending `Hook<T>` with a matching `HookState<T, Hook>` that exposes `initHook`, `didUpdateHook`, `build`, and `dispose` — mirroring `State`'s lifecycle almost exactly. Function hooks are the common case; the class form exists for hooks that need lifecycle callbacks of their own. Rousselet ships a broad standard library: primitives (`useEffect`, `useState`, `useMemoized`, `useRef`, `useCallback`, `useValueChanged`) and object-binding hooks that wrap Flutter/Dart types (`useStream`, `useFuture`, `useAnimationController`, `useTextEditingController`, `useScrollController`, `useTabController`, and many more), each handling creation and disposal.

Hot reload is the subtle part. Because state is keyed by position, the package overrides Flutter's default hot-reload behavior to preserve hook state across edits — but only up to the first changed line. Insert or remove a hook and every hook *after* the edit point is hard-reset during that reload (state fully recovers on the next full restart). This is documented behavior, not a bug, but it surprises newcomers who see a controller reset mid-session.

hooks_riverpod is the sanctioned integration path: `HookConsumerWidget` lets a single `build` method use both hooks and Riverpod providers, since the two libraries share an author and a philosophy of moving logic out of widget classes[^1].

## Production Notes

- **No compiler-enforced rules.** The two cardinal rules (call unconditionally, prefix with `use`) are conventions only. The community `flutter_hooks_lint` and Riverpod's `custom_lint`-based rules catch some violations, but there is no first-party guarantee comparable to React's ESLint plugin. Order bugs manifest as state binding to the wrong hook — hard to spot in review.
- **`useEffect` dependency arrays are a common footgun.** Omitting the keys list runs the effect on every build; passing `const []` runs it once; passing the wrong dependencies causes stale closures — the same class of bugs React developers hit, now in Dart where there is no lint to warn you.
- **Team idiom cost.** A codebase mixing `HookWidget` and `StatefulWidget` forces every contributor to know both models. Most Flutter hiring, docs, and third-party widget examples assume plain `StatefulWidget`, so onboarding carries a tax.
- **Testing.** Hooks can only run inside a building `HookWidget`, so unit-testing a custom hook means wrapping it in `HookBuilder` under `flutter_test` rather than calling it directly.
- **Maintenance cadence.** Releases are infrequent — 0.21.x has been the line since early 2025[^2]. The API is stable and null-safe, so this is generally fine, but do not expect rapid support for brand-new Flutter widgets; controller hooks sometimes lag SDK additions.
- **Still pre-1.0.** After seven years the package remains on 0.x. In practice it is production-stable and widely used, but the version number signals the author's reluctance to freeze the API rather than immaturity.

## When to Use / When Not

**Use when:**
- Your widgets are dense with controllers, animations, or stream/future subscriptions and the disposal boilerplate dominates.
- Your team already uses Riverpod and wants `HookConsumerWidget` to keep all logic in `build`.
- You have React background and the hooks mental model is already second nature.

**Avoid when:**
- The team is new to Flutter or values matching mainstream conventions and hiring expectations.
- The codebase is mostly simple stateless/stateful widgets where hooks add an idiom without removing much boilerplate.
- You cannot enforce the rules-of-hooks in CI and the team is large enough that convention drift is likely.

## Alternatives

- flutter/flutter `StatefulWidget` — the built-in baseline; more verbose but universally understood and lint-free of hook hazards. Use when convention and onboarding matter more than terseness.
- rrousselGit/riverpod — same author; a compile-safe state/dependency solution that reduces much of the same boilerplate without index-ordered rules. Use when the problem is shared state rather than per-widget lifecycle reuse.
- rrousselGit/provider — lighter dependency-injection over `InheritedWidget`; pairs with `StatefulWidget` instead of replacing it.
- felangel/bloc — event-driven state management with strong tooling and testability. Use when you want explicit, auditable state transitions over implicit hook state.
- rrousselGit/functional_widget — code-gen that turns functions into widgets; an alternate route to less boilerplate, composable with hooks.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-11 | Initial release — React hooks ported to Flutter[^4]. |
| 0.16.0-nullsafety | 2021-01-27 | Null-safety migration. |
| 0.18.0 | 2021-06-30 | Expanded hook library, API refinements. |
| 0.18.5+1 | 2022-06-24 | Late 0.18 maintenance line. |
| 0.21.2 | 2025-02-23 | Recent maintenance / new controller hooks. |
| 0.21.3+1 | 2025-08-19 | Latest published release as of this writing[^2]. |

## References

[^1]: flutter_hooks and Riverpod/Provider/freezed share a maintainer, Remi Rousselet (rrousselGit). https://github.com/rrousselGit
[^2]: pub.dev version history, flutter_hooks — latest 0.21.3+1 (2025-08). https://pub.dev/packages/flutter_hooks/versions
[^3]: Implementation principle — hooks stored as an indexed `List<Hook>` on the `Element`, replayed per build. flutter_hooks README, "Principle" section. https://github.com/rrousselGit/flutter_hooks#principle
[^4]: Motivation and origin — a Flutter implementation of React hooks (Dan Abramov, "Making sense of React Hooks"). https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889

## Tags

dart, flutter, hooks, state-management, react-hooks, widget-lifecycle, code-reuse, mobile, cross-platform, ui
