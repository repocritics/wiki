# reactiveui/ReactiveUI

> A functional-reactive MVVM framework for .NET that models UI state as observable streams rather than hand-wired property setters.

[GitHub repo](https://github.com/reactiveui/ReactiveUI) ·
[Official website](https://www.reactiveui.net) ·
[License: MIT](https://github.com/reactiveui/ReactiveUI/blob/main/LICENSE)

## Overview

ReactiveUI is a Model-View-ViewModel (MVVM) framework built on Reactive Extensions
(Rx). It began as `ReactiveXaml`, written by Paul (now Anaïs) Betts around 2011, and
is today a [.NET Foundation](https://dotnetfoundation.org) project maintained by a
small core team[^1]. The premise: instead of raising `INotifyPropertyChanged` events
by hand and coordinating updates imperatively, you express UI-state relationships as
`IObservable<T>` streams — `WhenAnyValue`, `ToProperty`, and `ReactiveCommand` — and
let the runtime propagate changes.

It is deliberately UI-toolkit-agnostic. A single core package (`ReactiveUI`, targeting
.NET Standard/`net8.0`) holds the reactive machinery, and thin platform packages
(`ReactiveUI.Wpf`, `ReactiveUI.WinForms`, `ReactiveUI.WinUI`, `ReactiveUI.Maui`,
`ReactiveUI.Blazor`, `ReactiveUI.Avalonia`, `ReactiveUI.Uno`, `ReactiveUI.AndroidX`)
supply the view-binding and activation glue for each host. This breadth is the selling
point and the maintenance burden: the same ViewModel code runs across desktop, mobile,
and web, but the project must track breaking changes across the entire Microsoft UI
matrix.

The defining tension is the learning curve. ReactiveUI asks developers to think in
observable sequences and to manage subscription lifetimes correctly. Teams that adopt
the mental model get testable, declarative ViewModels; teams that don't tend to fight
memory leaks and unreadable operator chains. It has always been a framework with a
steep on-ramp and a devoted, relatively small following rather than a default choice.

## Getting Started

```bash
dotnet add package ReactiveUI
dotnet add package ReactiveUI.Wpf   # or .Maui / .WinUI / .Avalonia / .Blazor
```

```csharp
using ReactiveUI;
using System.Reactive;

public class SearchViewModel : ReactiveObject
{
    private string _query = string.Empty;
    public string Query
    {
        get => _query;
        set => this.RaiseAndSetIfChanged(ref _query, value);
    }

    // ObservableAsPropertyHelper: a read-only property fed by a stream
    private readonly ObservableAsPropertyHelper<string> _result;
    public string Result => _result.Value;

    public ReactiveCommand<Unit, string> Search { get; }

    public SearchViewModel()
    {
        Search = ReactiveCommand.Create(() => $"Results for {Query}");
        _result = Search.ToProperty(this, x => x.Result);
    }
}
```

The View then binds to these members (`this.Bind`, `this.BindCommand`,
`this.OneWayBind`) inside a `WhenActivated` block so subscriptions are disposed when
the view goes away.

## Architecture / How It Works

The building blocks are small and composable:

- **`ReactiveObject`** — a base class implementing `INotifyPropertyChanged`;
  `RaiseAndSetIfChanged` is the setter helper that raises change notifications.
- **`WhenAnyValue`** — turns one or more properties into an `IObservable<T>` that emits
  on change. This is how derived state and validation are wired.
- **`ReactiveCommand`** — an `ICommand` that is itself an observable of its results,
  with a `CanExecute` stream and built-in `IsExecuting`/`ThrownExceptions` streams.
- **`ObservableAsPropertyHelper` (OAPH)** — projects an observable back into a
  bindable read-only property via `ToProperty`. The bridge from stream to XAML binding.
- **`WhenActivated`** — scopes subscriptions to a view's lifetime, the primary defense
  against the leaks that reactive code invites.

Routing (`RoutingState`, `IScreen`, `RoutedViewHost`) provides ViewModel-first
navigation, and a `Locator`-based service resolution (`IViewFor<T>`) maps ViewModels to
Views. Collection change-tracking has historically leaned on **DynamicData** (Roland
Pheasant's reactive collection library), which turns list mutations into observable
change-sets.

Recent versions restructured the dependency graph. ReactiveUI now ships in **two
interchangeable distributions with an identical public API**: the default packages run
on a new `ReactiveUI.Primitives` engine that drops the `System.Reactive` dependency
(surfacing `RxVoid`/`ISequencer`/`Signal<T>` instead of `Unit`/`IScheduler`/`Subject<T>`)
for a smaller closure and better trimming/AOT behavior, while the `*.Reactive` package
family keeps the classic `System.Reactive` types for drop-in interop[^2]. The
DynamicData-based routing/collection helpers moved into a separate `ReactiveUI.Routing`
package so the core no longer depends on DynamicData.

## Production Notes

- **Subscription lifetime is the #1 footgun.** An observable subscribed without
  disposal keeps its ViewModel (and often its View) alive. `WhenActivated` with a
  `DisposeWith(disposables)` discipline is not optional in real apps. Leaks here are
  the most common ReactiveUI production incident.
- **Debugging operator chains is hard.** A misbehaving `WhenAnyValue(...).Select(...)`
  pipeline produces no stack trace pointing at the offending frame. `.Log()` and
  `ThrownExceptions` subscriptions are the practical tools; errors inside a
  `ReactiveCommand` that aren't observed via `ThrownExceptions` are rethrown on the
  scheduler and can crash the app.
- **Scheduling and threading.** UI updates must land on the main-thread scheduler
  (`RxApp.MainThreadScheduler`). In unit tests you swap in `ImmediateScheduler` or the
  `ReactiveUI.Testing` `TestScheduler` (`With` / `AdvanceByMs`) to make time
  deterministic. Getting scheduler configuration wrong yields intermittent
  cross-thread exceptions.
- **AOT / trimming (iOS, MAUI, NativeAOT).** Historically the `Locator` reflection and
  Rx generics caused linker headaches. The Primitives distribution and
  `ReactiveUI.SourceGenerators` (which replace runtime reflection with generated
  boilerplate) exist specifically to improve the trimming/AOT story; migrating to them
  is the recommended path for size- and startup-sensitive apps.
- **Xamarin is gone.** As of the May 2024 Xamarin retirement, ReactiveUI removed legacy
  Xamarin targets; Xamarin.Forms apps must move to MAUI (`ReactiveUI.Maui`) and native
  Xamarin.Android/iOS to MAUI or `ReactiveUI.AndroidX`[^3].
- **Upgrade churn.** Because the framework spans WPF, WinUI, MAUI, Avalonia, Uno, and
  Blazor, major versions periodically drop or realign platform targets to follow the
  .NET release cadence. Read the release notes before bumping across a major.

## When to Use / When Not

**Use when:**
- You share ViewModel logic across multiple .NET UI toolkits (e.g. WPF + MAUI + Avalonia).
- Your UI is genuinely event-heavy — search-as-you-type, validation, throttling,
  combining async streams — where reactive operators simplify what would be tangled
  callback code.
- You value highly testable ViewModels and your team is comfortable with Rx.

**Avoid when:**
- The team is new to reactive programming and the app's interactions are mostly simple
  CRUD forms; the learning curve outweighs the benefit.
- You want the lightest possible MVVM with source-generated `INotifyPropertyChanged` and
  no Rx dependency — the CommunityToolkit MVVM is a smaller, simpler fit.
- You need a large ecosystem of first-party components and DI/navigation scaffolding out
  of the box; Prism is more batteries-included for enterprise line-of-business apps.

## Alternatives

- CommunityToolkit/dotnet — the MVVM Toolkit; source-generated `INotifyPropertyChanged`
  and relay commands. Use when you want lightweight MVVM without adopting Rx.
- PrismLibrary/Prism — modular MVVM with DI, navigation, and regions. Use when you need
  enterprise app composition rather than reactive composition.
- dotnet/reactive — Rx.NET (`System.Reactive`) itself. Use when you want reactive
  primitives without the MVVM/view-binding layer on top.
- reactivemarbles/DynamicData — reactive observable collections. Use alongside any MVVM
  stack when the hard part is reacting to collection changes, not property changes.
- Fody/PropertyChanged — IL-weaves `INotifyPropertyChanged` at build time. Use when you
  only want to eliminate boilerplate property setters, not adopt a full framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2010-06-11 | GitHub repository created (as ReactiveXaml)[^4]. |
| ReactiveUI 6 | ~2014 | Major rewrite; consolidated packaging and the modern API surface. |
| ReactiveUI 7 | ~2016 | Aligned with the modularized Rx.NET / `System.Reactive` NuGet packages. |
| — | 2024-05 | Legacy Xamarin targets removed following Xamarin retirement[^3]. |
| Current | 2026-07 (last push) | Dual distribution: `ReactiveUI.Primitives` (no System.Reactive) + `*.Reactive` interop; DynamicData split into `ReactiveUI.Routing`[^2]. |

## References

[^1]: ReactiveUI is a .NET Foundation project; current core team listed in the README (Glenn Watson, Chris Pulman, Rodney Littles II, Colt Bauman). https://github.com/reactiveui/ReactiveUI#core-team
[^2]: ReactiveUI README, "Choosing a distribution: ReactiveUI.Primitives or System.Reactive." https://github.com/reactiveui/ReactiveUI#readme
[^3]: ReactiveUI README, "Migration from Xamarin and .NET 8 MAUI"; Microsoft Xamarin support ended May 2024. https://github.com/reactiveui/ReactiveUI#migration-from-xamarin-and-net-8-maui
[^4]: GitHub API `repos/reactiveui/ReactiveUI`, `created_at` 2010-06-11. https://api.github.com/repos/reactiveui/ReactiveUI

## Tags

csharp, dotnet, mvvm, reactive-programming, functional-reactive-programming, rx, wpf, maui, avalonia, cross-platform, ui-framework
