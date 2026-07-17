# StephenCleary/Comparers

> A fluent builder for .NET `IComparer<T>` / `IEqualityComparer<T>` pairs that removes the boilerplate — and the correctness bugs — of hand-implementing comparison and equality.

[GitHub repo](https://github.com/StephenCleary/Comparers) ·
[NuGet: Nito.Comparers](https://www.nuget.org/packages/Nito.Comparers) ·
[License: MIT](https://github.com/StephenCleary/Comparers/blob/main/LICENSE)

## Overview

Comparers (shipped on NuGet as `Nito.Comparers`) is a small C# library by Stephen
Cleary — author of `Nito.AsyncEx` and a long-running .NET blog on async and
concurrency[^1]. It solves one narrow, perennial annoyance: correctly
implementing comparison in .NET means touching `IComparable<T>`, `IComparable`,
`IEquatable<T>`, `Object.Equals`, and `Object.GetHashCode`, and keeping all five
mutually consistent. Getting `GetHashCode` to agree with `Equals` is one of the
most common latent bugs in .NET domain code. Comparers replaces the whole ritual
with a fluent `ComparerBuilder.For<T>().OrderBy(...).ThenBy(...)` chain[^2].

The library's defining design choice is that **every comparer it produces also
implements equality**. A single object returned from the builder satisfies both
`IComparer<T>` and `IEqualityComparer<T>`, deriving `GetHashCode` from the same
key selectors used for ordering. That is convenient and eliminates a class of
consistency bugs, but it also means the equality semantics of a comparer are
whatever the ordering keys happen to be — a coupling worth understanding before
you drop one into a `Dictionary`.

It is mature and stable rather than actively developed. At ~450 stars it is a
respected niche utility, not a headline project, and the last commit landed in
December 2023[^3] — read that as "finished and in maintenance," not abandoned:
the surface area is small and the BCL contract it wraps has not changed.

## Getting Started

```bash
dotnet add package Nito.Comparers
```

```csharp
using Nito.Comparers;

// Build an IComparer<Person> that sorts by last name, then first name.
IComparer<Person> byName =
    ComparerBuilder.For<Person>()
                   .OrderBy(p => p.LastName)
                   .ThenBy(p => p.FirstName);

people.Sort(byName);

// The SAME object is also an IEqualityComparer<Person>:
var byNameEq = (IEqualityComparer<Person>)byName;
var dict = new Dictionary<Person, Address>(byNameEq);
```

To make a type comparable without hand-writing the five interface members,
derive from `ComparableBase<T>` and set `DefaultComparer` in a static
constructor; it wires up `Equals`, `GetHashCode`, and all comparison interfaces
for you. `EquatableBase<T>` is the equality-only counterpart.

## Architecture / How It Works

The core is `ComparerBuilder.For<T>()`, which returns a builder whose `OrderBy`
/ `ThenBy` (and `OrderBy(..., descending: true)`) operators each capture a key
selector and compose into an immutable comparer object. Composition is the whole
model: `Sequence()` lifts a `T` comparer into a lexicographic comparer over
`IEnumerable<T>`; `Null()` seeds an empty chain for runtime-built sorts;
`Default()` and `Reverse()` wrap existing comparers.

Two facts drive everything downstream:

- **Comparers are also equality comparers.** Each produced object implements
  `IFullComparer<T>` (the library's umbrella interface over `IComparer<T>` +
  `IEqualityComparer<T>` + the non-generic forms). `GetHashCode` is computed from
  the ordering key selectors, so equality is consistent with ordering *by
  construction*.
- **Base types are mixins via inheritance.** `ComparableBase<T>` /
  `EquatableBase<T>` implement the interface members and delegate to a static
  `DefaultComparer` you supply. This is a clean pattern, but it consumes your
  single base-class slot and depends on a static constructor running before the
  first comparison — an ordering concern in types with complex static state.

There is a companion `Nito.Comparers.Linq` package (pulled in by default) adding
key-selector overloads so anonymous types can be compared inline, e.g.
`items.Distinct(c => c.EquateBy(x => x.Surname))`. Distribution is via
`netstandard`, giving coverage across .NET Framework, .NET Core, .NET 5+, Mono,
and Unity without per-platform builds.

## Production Notes

- **The dual comparer/equality nature is a footgun if unexamined.** A comparer
  built to sort by `LastName` only, used as a `Dictionary` key comparer, treats
  two different people with the same last name as *equal keys*. This is internally
  consistent and documented, but it is not what a reader skimming the call site
  expects. Name the variable for its equality semantics, not just its sort order.
- **Fluent composition adds indirection.** Each `ThenBy` layer is a delegate
  invocation and the selectors box for value types in some paths. For a one-off
  sort this is irrelevant; inside a hot comparison loop over millions of elements,
  a hand-written `Comparison<T>` will beat it. Measure before using it on a
  performance-critical sort.
- **Reflection-based dynamic sorting is slow.** The README's "sort by property
  names at runtime" pattern calls `GetProperty(...).GetValue(...)` per comparison.
  Fine for building a comparer from user-selected columns; cache a compiled
  selector if the comparer runs in a tight loop.
- **Null handling differs from BCL defaults.** By default `null` sorts as "less
  than" everything; ordering nulls last, or giving nulls special treatment,
  requires the explicit `specialNullHandling` / boolean-key tricks. Read that
  section before assuming BCL-identical behavior.
- **Single-inheritance cost.** `ComparableBase<T>` occupies the base class slot;
  with an existing entity base class you fall back to assigning `DefaultComparer`
  and implementing the interfaces manually. A C# `record` may be the better fit
  when you control the type and only need equality.
- **Maintenance cadence.** No commits since late 2023[^3]. For a wrapper over
  stable BCL contracts this is low-risk, but do not expect rapid response to
  issues or new-framework fixes; budget for owning a fork if you depend on it.

## When to Use / When Not

**Use when:**
- You repeatedly write multi-key `IComparer`/`IEqualityComparer` implementations
  and want `Equals`/`GetHashCode` to be correct and consistent for free.
- You need comparers assembled at runtime (user-chosen sort columns, dynamic
  pipelines) with readable composition.
- You want one object usable for both sorting and hashing/dictionary keys.

**Avoid when:**
- The comparison is on a single field or two — the BCL's `Comparer<T>.Create`
  and a C# `record` cover it with no dependency.
- You're in a performance-critical sort loop where delegate indirection and
  allocation matter; hand-write the comparison.
- You need deep, structural, "what changed between two object graphs" comparison
  — that's a different problem (diffing), not ordering.

## Alternatives

- dotnet/runtime — the BCL's `Comparer<T>.Create(...)`, `EqualityComparer<T>.Default`, and `record` value-equality; use these when you need one or two simple comparisons and don't want a dependency.
- morelinq/MoreLINQ — key-selector LINQ operators like `DistinctBy`/`OrderBy` (some now native in .NET 6+); use when your real need is LINQ ergonomics over sequences, not reusable comparer objects.
- GregFinzer/CompareNETObjects — reflective deep object-graph comparison with a difference report; use when you need to know *what* differs between two instances, not to sort them.
- C# records / primary constructors — language-level value equality; use when you own the type, need equality only, and can adopt `record`.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2014-04-21 | Repository created; fluent `ComparerBuilder` core[^3]. |
| Nito.Comparers 6.x | current major | `netstandard`-based, LINQ extension package, `IFullComparer<T>` umbrella[^2]. |
| last commit | 2023-12-09 | Most recent change on `main`; library in maintenance[^3]. |

(Per-version release dates are not asserted here — verify against NuGet history
before citing a specific version date.)

## References

[^1]: Stephen Cleary — author site and blog (Nito libraries, async/concurrency). https://blog.stephencleary.com/
[^2]: Comparers README and documentation. https://github.com/StephenCleary/Comparers#readme
[^3]: GitHub repository metadata — created 2014-04-21, last push 2023-12-09, MIT license (fetched 2026-07). https://github.com/StephenCleary/Comparers

## Tags

csharp, dotnet, comparer, equality-comparer, icomparable, sorting, fluent-api, library, nuget, netstandard
