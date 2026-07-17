# CommunityToolkit/dotnet

> Microsoft's UI-agnostic .NET helper libraries — best known for the source-generator-driven MVVM Toolkit.

[GitHub repo](https://github.com/CommunityToolkit/dotnet) ·
[Documentation](https://learn.microsoft.com/dotnet/communitytoolkit/) ·
[License: MIT](https://github.com/CommunityToolkit/dotnet/blob/main/License.md)

## Overview

The .NET Community Toolkit is a set of four independent NuGet libraries — `CommunityToolkit.Common`, `CommunityToolkit.Diagnostics`, `CommunityToolkit.HighPerformance`, and `CommunityToolkit.Mvvm` — maintained by Microsoft under the .NET Foundation[^1]. They were split out of the older Windows Community Toolkit in 2021 specifically because they carry no UI-platform dependency: the same binaries run under WPF, WinUI, UWP, MAUI, Uno, Avalonia, Blazor, or a plain console app[^2]. The repo is C# and multi-targets down to `netstandard2.0`.

In practice this repo is the MVVM Toolkit. `CommunityToolkit.Mvvm` is the official successor to Laurent Bugnion's now-deprecated MvvmLight[^3], and it is the reason most people take a dependency here. Its defining choice is Roslyn source generators: attributes like `[ObservableProperty]` and `[RelayCommand]` generate the boilerplate `INotifyPropertyChanged` plumbing at compile time instead of at runtime via reflection. That makes it trim-safe and NativeAOT-friendly, which is the main axis on which it beats reflection-heavy frameworks like ReactiveUI or Prism.

The tradeoff is that the magic lives in generated code you do not see. Compile errors reference generated members, IDE tooling occasionally lags behind the generator, and the whole model depends on classes being declared `partial` — a requirement that trips up nearly every newcomer. The other three packages are smaller, stable utility libraries with a narrower audience.

## Getting Started

```bash
dotnet add package CommunityToolkit.Mvvm
```

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

// class MUST be partial — the generator emits into the other half
public partial class CounterViewModel : ObservableObject
{
    [ObservableProperty]              // generates a `Count` property + change notifications
    private int count;

    [RelayCommand]                    // generates an `IncrementCommand` (IRelayCommand)
    private void Increment() => Count++;
}
```

The generator turns the `count` field into a public `Count` property raising `PropertyChanged`, and turns `Increment` into an `IncrementCommand` you bind directly from XAML. No runtime reflection, no base-class ceremony beyond `ObservableObject`.

## Architecture / How It Works

Each of the four packages is standalone and versions in lockstep (they release together as 8.x). `CommunityToolkit.Mvvm` ships with **zero runtime NuGet dependencies** by design — it vendors everything it needs so a UI project does not inherit a transitive dependency graph.

- **MVVM Toolkit** — the runtime surface is small: `ObservableObject`, `ObservableRecipient`, `ObservableValidator`, `RelayCommand`/`AsyncRelayCommand`, and a messenger (`WeakReferenceMessenger.Default` / `StrongReferenceMessenger`). The interesting half is the source generators: `[ObservableProperty]`, `[RelayCommand]`, `[NotifyPropertyChangedFor]`, `[NotifyCanExecuteChangedFor]`, and validation attributes. These are incremental generators, so they re-run only for changed inputs, but they still expand your compile graph.
- **HighPerformance** — pooled buffer types (`MemoryOwner<T>`, `SpanOwner<T>`, `ArrayPoolBufferWriter<T>`), a lock-free `StringPool`, `Ref<T>`/`NullableRef<T>`/`Box<T>`, `BitHelper`, and 2D `Memory2D<T>`/`Span2D<T>` supporting discontiguous regions. Used in Paint.NET[^2].
- **Diagnostics** — `Guard` (e.g. `Guard.IsNotNull(x)`) and `ThrowHelper`, which move `throw` statements into separate non-inlined methods so the JIT keeps hot paths small.
- **Common** — a thin set of shared helpers the other libraries lean on.

Because `[ObservableProperty]` historically generated a property *from a field*, the field name drives the generated property name (`count` → `Count`), and the class must be `partial`. Version 8.4 added support for putting `[ObservableProperty]` on a `partial property` instead of a field, but that path requires the C# 13 compiler (.NET 9 SDK)[^4].

## Production Notes

- **`partial` is mandatory.** The single most common error is "the generated members don't exist" — almost always because the class (or an enclosing type) isn't declared `partial`. Nested view models need every enclosing class partial too.
- **Old SDKs break the generators.** Source generators require a modern Roslyn. Building 8.x with an old `dotnet` SDK or a stale MSBuild produces confusing "type or member not found" errors rather than a clean message. Pin the SDK in `global.json`.
- **`WeakReferenceMessenger` is the default and it can silently drop messages.** If the only reference to a recipient is the messenger registration, the recipient gets garbage-collected and stops receiving. "My handler stopped firing" usually means the view model was collected; switch to `StrongReferenceMessenger` or hold a reference deliberately.
- **IoC is no longer bundled.** Earlier versions shipped a `Microsoft.Toolkit.Mvvm.DependencyInjection.Ioc` helper; current guidance is to use `Microsoft.Extensions.DependencyInjection` directly. Docs and blog posts predating 8.0 reference the removed API.
- **`Guard` is largely redundant on modern .NET.** `ArgumentNullException.ThrowIfNull` and friends (added in .NET 6+) cover most of what the Diagnostics package offered; new code on current runtimes rarely needs it.
- **Trimming / NativeAOT is the payoff.** Because the MVVM Toolkit avoids reflection, it survives aggressive trimming and AOT where reflection-based MVVM stacks emit trim warnings or fail at runtime — the main reason to prefer it in MAUI/AOT scenarios.

## When to Use / When Not

**Use when:**
- You want MVVM primitives that work identically across every .NET UI stack.
- You target NativeAOT, trimming, or MAUI and need reflection-free view models.
- You want to delete `INotifyPropertyChanged` boilerplate without adopting a full application framework.
- You need pooled buffers / `Span2D` (HighPerformance) or inline-friendly guards (Diagnostics) as small, dependency-free utilities.

**Avoid when:**
- You want a full app framework with navigation, regions, and modularity — this is libraries, not scaffolding.
- Your team is hostile to source generators or stuck on an SDK too old to run them.
- You prefer Rx/observable-first view models — ReactiveUI fits that mental model better.
- You only need one guard clause or one pooled array; the BCL now covers much of that natively.

## Alternatives

- reactiveui/reactiveui — use when you want Rx/observable-driven view models and accept a steeper model plus some reflection.
- PrismLibrary/Prism — use when you need a full application framework (modules, regions, navigation), not just MVVM primitives.
- CommunityToolkit/Maui — sibling toolkit; use when you need MAUI-specific controls, behaviors, and converters on top of these primitives.
- dotnet/runtime — for the HighPerformance overlap, use `System.Buffers`/`ArrayPool<T>` and `Span<T>` directly when you'd rather not add a dependency.
- MvvmLight — the deprecated predecessor; only relevant when migrating an old codebase off it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 7.x | 2021 | Last line published as `Microsoft.Toolkit.Mvvm`, inside the Windows Community Toolkit. |
| — | 2021-10 | Libraries split into this standalone `CommunityToolkit/dotnet` repo[^2]. |
| 8.0 | 2022 | Renamed to `CommunityToolkit.Mvvm`; Roslyn source generators (`[ObservableProperty]`, `[RelayCommand]`) introduced[^3]. |
| 8.1 | 2022 | Generator and analyzer refinements. |
| 8.2 | 2023 | Additional notification/validation generator attributes. |
| 8.3 | 2024 | Incremental-generator performance and diagnostics improvements. |
| 8.4 | 2024 | `[ObservableProperty]` on `partial property` (requires C# 13 / .NET 9 SDK)[^4]. |

## References

[^1]: Repository description and .NET Foundation membership — https://github.com/CommunityToolkit/dotnet
[^2]: README, ".NET Community Toolkit" — origin as a split from the Windows Community Toolkit, cross-platform intent, Paint.NET usage. https://github.com/CommunityToolkit/dotnet/blob/main/README.md
[^3]: MVVM Toolkit documentation — "official successor of MvvmLight", source-generator model. https://learn.microsoft.com/dotnet/communitytoolkit/mvvm/
[^4]: MVVM Toolkit `[ObservableProperty]` on partial properties. https://learn.microsoft.com/dotnet/communitytoolkit/mvvm/generators/observableproperty

## Tags

csharp, dotnet, mvvm, source-generators, maui, wpf, winui, high-performance, nativeaot, dotnet-foundation
