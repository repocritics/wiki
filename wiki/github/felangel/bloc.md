# felangel/bloc

> A predictable state management library for Dart and Flutter built on the BLoC (Business Logic Component) pattern and Dart streams.

[GitHub repo](https://github.com/felangel/bloc) ·
[Official website](https://bloclibrary.dev) ·
[License: MIT](https://github.com/felangel/bloc/blob/master/LICENSE)

## Overview

Bloc is the reference implementation of the BLoC pattern, a state-management
convention Google engineers introduced at DartConf 2018 to separate presentation
from business logic in a way that works identically on Flutter and AngularDart[^1].
Felix Angelov's library, first published in late 2018, turned that talk into a
maintained package family. As of 2026 it is one of the two or three most-used
Flutter state-management options, alongside Provider/Riverpod, and the one most
associated with large, testable, team-scale codebases.

The core idea is a unidirectional data flow: inputs (events, or in the simpler
`Cubit` form, method calls) go in, an immutable stream of states comes out, and
the UI is a pure function of the latest state. This buys predictability, an
observable audit trail of every transition (`BlocObserver`), and first-class
testability — at the cost of boilerplate. Defining an event class, a state class,
and a handler for a single interaction is the standing complaint, and the reason
`Cubit` (methods instead of events) was added in 2020 as a lighter on-ramp.

Bloc is a monorepo of ~9 packages: `bloc` (pure-Dart core), `flutter_bloc`
(widgets), `bloc_test`, `hydrated_bloc` (persistence), `replay_bloc`,
`bloc_concurrency` (event transformers), `bloc_tools` and `bloc_lint`
(tooling/lints), and `angular_bloc`. Most apps depend only on `flutter_bloc`,
which re-exports the core.

## Getting Started

```bash
flutter pub add flutter_bloc   # pulls in bloc; use `dart pub add bloc` for pure Dart
```

The simpler `Cubit` API — no event classes:

```dart
import 'package:bloc/bloc.dart';

class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);      // initial state
  void increment() => emit(state + 1);
}
```

The full `Bloc` API — events mapped to states via `on<Event>` handlers:

```dart
import 'package:bloc/bloc.dart';

sealed class CounterEvent {}
class Incremented extends CounterEvent {}

class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0) {
    on<Incremented>((event, emit) => emit(state + 1));
  }
}
```

Wiring into Flutter:

```dart
BlocProvider(
  create: (_) => CounterCubit(),
  child: BlocBuilder<CounterCubit, int>(
    builder: (context, count) => Text('$count'),
  ),
)
// dispatch: context.read<CounterCubit>().increment();
```

## Architecture / How It Works

Everything sits on Dart `Stream`s. `BlocBase` owns a state and a broadcast
stream; `emit(newState)` pushes to it and `BlocBuilder`/`BlocListener` subscribe.
`Cubit` exposes `emit` directly through methods; `Bloc` adds an event pipeline on
top: dispatched events flow through an **event transformer** into the registered
`on<Event>` handlers, which then call `emit`.

Key internals and their consequences:

- **State equality gate.** `emit` is a no-op when the new state `==` the current
  state. This is why Bloc code pairs immutable state with `Equatable` (or
  `freezed`) so equal states are deduplicated. Get this wrong in either direction
  and you get missing rebuilds or redundant ones.
- **Event transformers** (`bloc_concurrency`) decide how concurrent events are
  processed: `concurrent()` (the default — events run in parallel, order not
  guaranteed), `sequential()`, `droppable()`, and `restartable()`. Choosing the
  wrong one is a common source of race conditions in async handlers.
- **`BlocObserver`** is a global hook fired on every `onCreate`, `onEvent`,
  `onChange`, `onTransition`, `onError`, and `onClose` — the basis for logging,
  analytics, and debugging.
- **`flutter_bloc` widgets** are built over the `provider` package's
  `InheritedWidget` propagation. `BlocProvider` injects and disposes the instance;
  `BlocBuilder` rebuilds on state changes (narrowed by `buildWhen`); `BlocListener`
  runs side effects (`listenWhen`); `BlocConsumer` combines both;
  `context.select` rebuilds on a derived slice only.
- **`hydrated_bloc`** transparently persists and restores state by overriding
  `fromJson`/`toJson`, backed by a pluggable `HydratedStorage`.

The `Bloc` vs `Cubit` split is the defining design tension: `Cubit` removes the
event indirection (less code, but no event stream to observe or transform), while
`Bloc` keeps the full audit trail and concurrency control. Teams routinely mix
both in one app.

## Production Notes

- **Boilerplate is real.** Event + state classes per feature add up. Codegen
  (`freezed`, `bloc` VS Code/IntelliJ extensions) mitigates it but does not
  remove it. If a screen has trivial local state, a `Cubit` — or plain
  `ValueNotifier` — is usually the right call over a full `Bloc`.
- **"emit was called after the bloc/cubit was closed."** The most common runtime
  error: an async handler awaits, the widget/bloc is disposed, then `emit` fires.
  Guard long async work with `if (isClosed) return;` or `emit.isDone`, and prefer
  event transformers over fire-and-forget async in handlers.
- **`Equatable` on state is effectively mandatory.** Without value equality,
  every `emit` of a structurally-identical state triggers a rebuild; with a
  mutable state object you can mutate in place and have `emit` silently skipped by
  the equality gate.
- **Concurrency defaults surprise people.** The default `concurrent` transformer
  means two rapid dispatches of the same event can interleave. Search/typeahead
  and submit flows almost always want `restartable()` or `droppable()`.
- **Testing is a genuine strength.** `bloc_test`'s `blocTest` helper (arrange →
  `act` → assert `expect: [states]`) makes state transitions cheap to test, and
  is one of the main reasons large teams pick Bloc.
- **The v8 migration was disruptive.** The move from the `mapEventToState`
  generator override to `on<Event>` handler registration (deprecated in the 7.2
  line, removed in 8.0) rewrote how every Bloc is authored[^2]. Older tutorials
  and Stack Overflow answers still show the removed API — a frequent trap for
  newcomers.
- **`hydrated_bloc` needs migration discipline.** Persisted JSON must stay
  backward-compatible or `fromJson` will throw on old payloads after a state-shape
  change; version your serialization and handle decode failures.

## When to Use / When Not

**Use when:**
- The app is large or team-owned and you want an enforced, uniform state pattern.
- You value an observable, testable, replayable state history (auditing, undo,
  analytics via `BlocObserver`).
- You need the same logic to run across Flutter and Dart/AngularDart targets.
- Complex async flows benefit from explicit concurrency control per event.

**Avoid when:**
- The app or screen is small; the ceremony outweighs the payoff (use `Cubit`
  alone, `ValueNotifier`, or `setState`).
- Your team prefers implicit reactivity / dependency injection over explicit
  event–state plumbing (Riverpod, MobX fit better).
- You want minimum boilerplate above all and accept more framework opinionation
  (GetX).

## Alternatives

- rrousselet/riverpod — compile-safe provider evolution with less boilerplate; use when you want reactive DI without writing event/state classes.
- rrousselet/provider — the lower-level `InheritedWidget` wrapper Bloc itself builds on; use for simple dependency injection and value exposure.
- jonataslaw/getx — bundles state, routing, and DI; use when you want minimal code and accept the all-in-one, magic-heavy tradeoff.
- mobxjs/mobx.dart — observables + reactions; use when you prefer implicit, fine-grained reactivity over explicit events.
- Plain `ValueNotifier` / `ChangeNotifier` — use when state is local and trivial and a library is overkill.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2018-11 | Initial release; streams + `mapEventToState`[^1]. |
| 6.0 | 2020-08 | `Cubit` introduced as a lighter, event-free API; `Bloc` refactored on top of `Cubit`. |
| 7.0 | 2021-03 | Null-safety migration; API cleanup. |
| 7.2 | 2021 | `on<Event>` handler API added; `mapEventToState` deprecated. |
| 8.0 | 2022 | `mapEventToState` removed; event transformers via `bloc_concurrency`[^2]. |
| 9.0 | 2024–2025 | Later major line; tooling additions (`bloc_lint`, `bloc_tools`). |

## References

[^1]: BLoC pattern origin — Paolo Soares & Cong Hui, "Flutter / AngularDart – Code sharing, better together," DartConf 2018. https://www.youtube.com/watch?v=PLHln7wHgPE
[^2]: Bloc migration guide (`mapEventToState` → `on<Event>`, 7.x → 8.0). https://bloclibrary.dev/migration

## Tags

dart, flutter, state-management, bloc-pattern, streams, reactive, mobile, angulardart, testability, cubit
