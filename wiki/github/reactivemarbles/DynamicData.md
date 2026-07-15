# reactivemarbles/DynamicData

> Reactive Extensions for collections: express filter/sort/transform/group/bind
> as observable pipelines over a mutating in-memory data source.

[GitHub repo](https://github.com/reactivemarbles/DynamicData) ·
[NuGet](https://www.nuget.org/packages/DynamicData) ·
[License: MIT](https://github.com/reactivemarbles/DynamicData/blob/main/LICENSE)

## Overview

DynamicData applies the Reactive Extensions (Rx.NET) model to collections. Plain
Rx gives you `IObservable<T>` — a stream of values — but nothing for the common
case where you hold a *collection* that is mutated over time (items added,
updated, removed) and you want derived views that stay in sync. DynamicData fills
that gap with the *observable change set*: `IObservable<IChangeSet<T>>`, a stream
of deltas rather than a stream of whole-collection snapshots. Operators consume
change sets and emit change sets, so a filter, sort, transform, or UI binding
only reprocesses the items that actually changed[^1].

It was written by Roland Pheasant, originally under `RolandPheasant/DynamicData`,
and later moved into the `reactivemarbles` organization alongside its principal
consumer, ReactiveUI[^2]. In practice DynamicData is the collection layer of the
.NET MVVM/Rx stack: a `SourceCache` or `SourceList` is the model, a pipeline of
operators is the view-model logic, and `.Bind(out var collection)` produces a
`ReadOnlyObservableCollection<T>` that a WPF/Avalonia/MAUI/Uno view binds to.

The defining tension is conceptual weight against payoff. The change-set model is
genuinely different from LINQ-over-collections, and the two source types
(`SourceCache<TObject,TKey>` keyed, `SourceList<TObject>` unkeyed) plus 60-plus
operators are a real learning curve. In exchange, code that would otherwise be
tangled manual add/update/remove bookkeeping collapses into a declarative
pipeline that maintains itself.

## Getting Started

```bash
dotnet add package DynamicData
```

```csharp
using DynamicData;
using DynamicData.Binding;

// A keyed source: TObject = Trade, TKey = long
var trades = new SourceCache<Trade, long>(t => t.Id);

ReadOnlyObservableCollection<TradeProxy> live;

var subscription = trades.Connect()               // IObservable<IChangeSet<Trade,long>>
    .Filter(t => t.Status == TradeStatus.Live)    // only live trades
    .Transform(t => new TradeProxy(t))            // map to a view model
    .SortBy(p => p.Timestamp)                      // keep ordered
    .Bind(out live)                                // maintain a bindable collection
    .DisposeMany()                                 // dispose proxies when removed
    .Subscribe();

// Later mutations flow through automatically:
trades.AddOrUpdate(newTrade);   // 'live' updates itself if the trade is Live
```

Mutating a source outside a batch emits one notification per call; wrap multiple
edits in `source.Edit(inner => { ... })` to collapse them into a single change
set. Expose a source read-only with `.AsObservableCache()` / `.AsObservableList()`
to hide the edit methods from consumers.

## Architecture / How It Works

The core abstraction is `IChangeSet`, an enumerable of `Change` records — each
carrying a reason (Add, Update, Remove, Refresh, Moved) and the affected item(s).
Every operator is a function `IObservable<IChangeSet<T>> -> IObservable<IChangeSet<T>>`
that maintains internal state and translates incoming changes into outgoing ones.
`Filter`, for example, tracks the currently-passing set and emits only the
adds/removes that cross the predicate boundary. Because everything is a delta, a
large collection with one changed element does proportionally small downstream
work, not a full re-scan.

There are two source families, and choosing between them is the first real
design decision:

- **`SourceCache<TObject,TKey>`** — keyed, dictionary-backed. Updates replace by
  key; identity is the key, not object reference. This is the common choice for
  domain entities that have an id.
- **`SourceList<TObject>`** — ordered, index-based, no key. Supports duplicates
  and positional operations. Required when order or duplicates matter, but its
  operators are a slightly smaller set than the cache's.

Sorting has two implementations that trip people up: the older `Sort` (with
`SortExpressionComparer`) and the newer `SortAndBind` / `SortBy` family that
fuses sorting and binding for better performance on large or frequently-changing
sets[^3]. `Bind(out collection)` is the terminal step that converts the change
stream back into a mutating `ObservableCollection`-shaped target emitting
`INotifyCollectionChanged` for the UI — but it marshals nothing itself, so you
place `ObserveOn(scheduler)` upstream to reach the UI thread. `AutoRefresh`
bridges `INotifyPropertyChanged` so item property changes re-trigger
`Filter`/`Sort`, and `DisposeMany` handles lifetimes when `Transform` produces
disposables.

## Production Notes

- **Threading is your responsibility.** DynamicData does not serialize access to
  a source. `SourceCache`/`SourceList` edits are internally locked, but a pipeline
  that reads a source from multiple threads without an explicit scheduler will
  interleave. The common bug is mutating a source from a background thread and
  binding without `ObserveOn(RxApp.MainThreadScheduler)` upstream of `Bind`,
  producing cross-thread UI exceptions.
- **`ToObservableChangeSet` grows unbounded.** Building a change set from a raw
  `IObservable<T>` caches every item forever unless you pass `limitSizeTo:` or
  `expireAfter:`. This is a frequent memory-leak source.
- **`AutoRefresh` cost scales with item count.** It subscribes to every element's
  `PropertyChanged`; on large collections this is measurable. Prefer refreshing on
  a specific property (`AutoRefresh(x => x.Status)`) or restructuring so the
  source is re-`AddOrUpdate`d instead.
- **Subscriptions leak if not disposed.** Every `.Subscribe()` and every
  `out`-bound collection holds resources. Store the `IDisposable` (typically in a
  `CompositeDisposable`) and dispose it with the view model.
- **Collections don't carry Refresh.** A `ReadOnlyObservableCollection` cannot
  represent a Refresh notification; when using `AutoRefresh`, prefer
  `BindToObservableList(out ...)` over binding to a collection for derived views.
- **Debugging pipelines is opaque.** A misbehaving chain yields a wrong final
  collection with little signal about which operator is at fault; `.Do(...)` taps
  and intermediate `AsObservableCache()` snapshots are the practical tools.

## When to Use / When Not

**Use when:**
- You have an in-memory collection that mutates over time and one or more derived
  views (filtered/sorted/grouped/paged) that must stay in sync.
- You're building MVVM on WPF/Avalonia/Uno/MAUI and want view models to expose
  self-maintaining `ReadOnlyObservableCollection`s, especially with ReactiveUI.
- You already think in Rx and want the same composition model for collections.

**Avoid when:**
- The data is static or refetched wholesale — a plain list plus `CollectionView`
  is simpler and has no learning curve.
- Your team isn't comfortable with Rx; the change-set model amplifies Rx's
  debugging difficulty.
- You need a persistent or distributed store — DynamicData is in-memory only;
  it's a query/projection layer, not a database.

## Alternatives

- ReactiveUI/ReactiveUI — parent ecosystem; its `ReactiveList` is deprecated in
  favor of DynamicData, so pair them rather than pick between them.
- dotnet/reactive (Rx.NET) — use directly when you have streams of values, not
  collections needing delta projection.
- Use ObservableCollections (Cysharp) instead when you want a lighter-weight
  observable-collection library with LINQ-style views and less Rx conceptual load.
- Use LINQ + CollectionViewSource instead when the collection is small, mostly
  static, and only needs UI-side filtering/sorting.
- Use a real database (SQLite/LiteDB) instead when data must persist or exceed
  memory; DynamicData can project a query result but not own the storage.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-11-21 | First commit as `RolandPheasant/DynamicData`[^2]. |
| 6.x | ~2019–2020 | Broad operator maturity; ReactiveUI adoption. |
| 7.0 | ~2021 | .NET 5/6 targeting era; ongoing 7.x line through 2023. |
| 8.0 | 2023-09-22 | Major line; nullable annotations, API cleanup[^4]. |
| 9.0 | 2024-07-17 | Current major; `SortAndBind` improvements, trimming/AOT work[^4]. |
| 9.4.x | 2025–2026 | Active maintenance; 9.5 in preview as of mid-2026[^4]. |

## References

[^1]: DynamicData README — observable change sets and the source/operator model.
      https://github.com/reactivemarbles/DynamicData/blob/main/README.md
[^2]: Roland Pheasant's DynamicData blog and original repository history.
      https://dynamic-data.org/
[^3]: DynamicData binding and sorting operators (`Bind`, `SortAndBind`).
      https://github.com/reactivemarbles/DynamicData/tree/main/src
[^4]: DynamicData GitHub releases (8.0.1 2023-09, 9.0.1 2024-07, 9.4.x 2025–2026).
      https://github.com/reactivemarbles/DynamicData/releases

## Tags

csharp, dotnet, reactive-extensions, rx-net, observable-collections, mvvm,
reactiveui, change-tracking, in-memory-data, reactive-programming
