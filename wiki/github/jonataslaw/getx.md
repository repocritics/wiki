# jonataslaw/getx

> An all-in-one Flutter package that bundles state management, dependency injection, and context-less navigation behind a single `Get` facade.

[GitHub repo](https://github.com/jonataslaw/getx) ·
[pub.dev/packages/get](https://pub.dev/packages/get) ·
[License: MIT](https://github.com/jonataslaw/getx/blob/master/LICENSE)

## Overview

GetX (published on pub.dev as `get`) is a Flutter microframework by Jonatas Borges, first published in 2019[^1]. It packages three normally-separate concerns — reactive state management, dependency injection, and routing — into one library reached through a global `Get` object. Its selling point is boilerplate reduction: making a value reactive is `.obs`, rebuilding a widget is `Obx(() => ...)`, navigating is `Get.to(Page())` with no `BuildContext`. For a period around 2020–2022 it was the most-liked package on pub.dev and a default choice for teams optimizing for delivery speed.

The defining tension is exactly that breadth. GetX deliberately violates the Flutter idiom that navigation, theming, and localization flow through `BuildContext` and the widget tree; instead it holds global singletons and a global navigator key. That is what makes the API terse, and it is also the source of most criticism: it couples an app to one large dependency, encourages patterns (service locators, global mutable state) that are hard to test and reason about, and blurs the line between "state manager" and "framework." The Flutter community is unusually polarized on it — heavily used in tutorials and production apps, yet frequently discouraged in architecture guidance.

Maintenance is concentrated on a single primary author, and release cadence has slowed markedly since the 4.x line. As of 2026 the repo still sees commits but issues accumulate faster than they close (open issue count is in the low thousands), and the long-promised 5.0 rework has been in progress for an extended period[^2]. Treat GetX as mature-but-slowing rather than actively evolving.

## Getting Started

```yaml
# pubspec.yaml
dependencies:
  get: ^4.6.6
```

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

// 1. Business logic in a controller
class CounterController extends GetxController {
  final count = 0.obs;            // reactive integer
  void increment() => count++;
}

void main() => runApp(
  // GetMaterialApp is only required if you use routing/snackbars/i18n
  GetMaterialApp(home: HomePage()),
);

class HomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final c = Get.put(CounterController());   // register + instantiate
    return Scaffold(
      appBar: AppBar(title: Obx(() => Text('Clicks: ${c.count}'))),
      body: Center(
        child: ElevatedButton(
          onPressed: () => Get.to(() => DetailPage()),  // no context
          child: const Text('Go to detail'),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: c.increment, child: const Icon(Icons.add)),
    );
  }
}

class DetailPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final c = Get.find<CounterController>();   // retrieve existing instance
    return Scaffold(body: Center(child: Obx(() => Text('${c.count}'))));
  }
}
```

Only `Get.put`/`Get.find` and `Obx` require nothing beyond a plain `MaterialApp`; `GetMaterialApp` is needed once you use `Get.to`, dialogs, snackbars, or translations.

## Architecture / How It Works

GetX is three loosely-independent subsystems that share one namespace. Each is tree-shaken separately — using only state management does not pull routing into the build.

- **Reactive state** — `.obs` wraps a value in an `Rx<T>` observable. `Obx`/`GetX` widgets subscribe by *reading* observables inside their builder: the widget records which `Rx` values it touched during build and rebuilds only when those specific values change. This is a fine-grained observer pattern implemented without `Stream` or `ChangeNotifier`. There is also a second, non-reactive `GetBuilder` that rebuilds on an explicit `update()` call — lighter but manual.
- **Dependency injection** — `Get.put`/`Get.lazyPut`/`Get.find` are a global service locator keyed by type (and optional `tag`). Instances default to lazy creation and to automatic disposal when the route that created them is removed, unless marked `permanent: true`. `Bindings` classes declare which dependencies a route needs so DI is wired at navigation time.
- **Routing** — `Get.to`/`Get.toNamed`/`Get.off`/`Get.back` drive a global `Navigator` via an internal `GlobalKey<NavigatorState>` that `GetMaterialApp` installs. This is what makes context-free navigation possible.

The controller lifecycle is `GetLifeCycle` (`onInit`, `onReady`, `onClose`) rather than the widget lifecycle, so business logic is decoupled from `StatefulWidget`. The cost of the design is the global surface: the navigator key, the DI registry, and `Get.locale`/`Get.theme` are process-wide singletons. Convenience and testability are traded against each other deliberately.

## Production Notes

- **Testing friction.** Because DI and navigation are global singletons, tests must reset state between cases (`Get.reset()`) and often need `Get.testMode = true`. Widget tests that assume the standard `Navigator`/`BuildContext` flow do not directly apply. This is the most common real-world complaint.
- **`Obx` "improper use" errors.** `Obx` only rebuilds if an observable is *read during its build*. A very common bug is building an `Obx` whose body reads no `.obs` value (e.g. the read is inside a callback), which throws at runtime with an "improper use of GetX" message. The mental model — subscription happens by reading — has to be learned.
- **Memory and disposal.** Automatic disposal is tied to route removal. Controllers put outside a route, marked `permanent: true`, or created via `Get.putAsync` can outlive their intended scope and leak. `GetxService` is the intended escape hatch for app-lifetime singletons.
- **Global mutable state.** Nothing prevents any code from calling `Get.find()` anywhere, which erodes explicit dependency graphs and makes data flow hard to trace in large teams. Several widely-shared Flutter architecture guides recommend against GetX for exactly this reason; it is a real, contested tradeoff rather than a bug.
- **Single-package coupling.** Adopting GetX for state tends to pull in its routing and snackbar/dialog helpers too, because they are convenient and already present. Migrating off later touches navigation, DI, and UI-feedback code at once.
- **Maintenance risk.** Bus factor is effectively one, PR turnaround is slow, and the 5.0 rewrite has been long-running[^2]. For new projects, weigh this against alternatives with broader maintainer bases before committing.

## When to Use / When Not

**Use when:**
- You want minimum boilerplate and fast delivery, and one dependency covering state + DI + routing is a feature, not a smell.
- Small-to-medium apps or solo/prototyping work where global convenience outweighs strict testability.
- You specifically want context-free navigation, dialogs, and snackbars.

**Avoid when:**
- The codebase is large or multi-team and you need explicit, traceable dependency graphs and first-class testability.
- Your team follows idiomatic Flutter architecture guidance (scoped providers, `BuildContext`-driven navigation).
- You want a library with a broad maintainer base and predictable release cadence.
- You only need one concern (just state, or just DI) — a focused package is a smaller commitment.

## Alternatives

- fluttercommunity/provider — the community-endorsed baseline; `InheritedWidget` wrapper, scoped and testable. Use when you want idiomatic, minimal state sharing.
- rrousselGit/riverpod — provider's successor by the same author; compile-safe, no `BuildContext` needed to read, better testing. Use when you want provider's model without its runtime footguns.
- felangel/bloc — event/state streams with strict, testable structure. Use when you want enforced architecture and traceable state transitions on larger teams.
- Signals / `flutter_hooks` + a router — pick a focused reactive primitive plus go_router. Use when you want GetX's ergonomics without one monolithic dependency.
- flutter/packages (go_router) — official routing only. Use when your quarrel with GetX is specifically its navigation coupling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2019-11 | Initial pub.dev publication as `get`[^1]. |
| 2.0 | 2020 | API reshaped; documented "breaking changes from 2.0" baseline. |
| 3.x | 2020 | Reactive `Rx`/`Obx`, bindings, DI maturation; rapid popularity growth. |
| 4.0 | 2021 | Major line: null-safety, `GetConnect`, refined routing. |
| 4.6.x | 2022 | Current stable line; subsequent releases mostly maintenance[^3]. |
| 5.0 | in progress | Long-running rework, not yet stabilized as of 2026[^2]. |

## References

[^1]: `get` package on pub.dev — publisher, versions, and metrics. https://pub.dev/packages/get
[^2]: GetX 5.0 tracking / ongoing rewrite discussion in the repository issues. https://github.com/jonataslaw/getx/issues
[^3]: `get` changelog on pub.dev. https://pub.dev/packages/get/changelog

## Tags

dart, flutter, state-management, dependency-injection, routing, navigation, reactive, mobile, framework, service-locator
