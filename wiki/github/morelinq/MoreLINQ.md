# morelinq/MoreLINQ

> Extra extension methods for LINQ to Objects, filling the gaps the .NET base class library leaves — and increasingly overlapping with it.

[GitHub repo](https://github.com/morelinq/MoreLINQ) ·
[Official website](https://morelinq.github.io/) ·
[License: Apache-2.0](https://github.com/morelinq/MoreLINQ/blob/master/COPYING.txt)

## Overview

MoreLINQ is a C# library that adds operators to LINQ to Objects — methods like
`Batch`, `Pairwise`, `MaxBy`/`Maxima`, `DistinctBy`, `Scan`, `WindowLeft`,
`Cartesian`, `FullJoin`, and roughly a hundred others — in the same lazy,
`IEnumerable<T>`-based style as `System.Linq`. It is one of the oldest and
most-depended-on LINQ helper libraries in the .NET ecosystem; the project
predates its 2015 GitHub home, having originated on Google Code years earlier[^1].
Atif Aziz has been the long-running lead maintainer, with Jon Skeet among the
early contributors[^1].

The defining tension for MoreLINQ in 2026 is that the base class library keeps
catching up to it. .NET 6 added `DistinctBy`, `MinBy`/`MaxBy`, and `Chunk` (the
BCL's answer to `Batch`) directly to `System.Linq`, and earlier framework
versions had already added `Zip`, `Append`, `Prepend`, `TakeLast`, and
`SkipLast`[^2]. Each such addition creates a name/signature collision with the
MoreLINQ equivalent, forcing the library to deprecate operators (`MaxBy` →
`Maxima`), and pushing consumers toward selective imports. MoreLINQ remains
valuable for the operators the BCL still lacks, but its surface is slowly being
commoditized from underneath.

## Getting Started

```bash
dotnet add package morelinq
```

```csharp
using MoreLinq;

var nums = new[] { 1, 5, 2, 8, 3 };

// Batch into fixed-size chunks (lazy)
foreach (var chunk in nums.Batch(2))
    Console.WriteLine(string.Join(",", chunk));   // "1,5"  "2,8"  "3"

// Maxima: ALL elements tied for the maximum projected key
var longest = new[] { "a", "bb", "cc", "d" }.Maxima(s => s.Length); // "bb","cc"

// Pairwise: apply a function to each element and its predecessor
var deltas = nums.Pairwise((a, b) => b - a);       // 4, -3, 6, -5
```

To avoid collisions with identically named BCL methods, import only the
operators you need using C# `using static` (available since MoreLINQ 3.0)[^3]:

```csharp
using static MoreLinq.Extensions.LagExtension;
using static MoreLinq.Extensions.LeadExtension;
using MoreEnumerable = MoreLinq.MoreEnumerable;   // alias for generator methods
```

## Architecture / How It Works

MoreLINQ is a pure library — no runtime services, no dependencies beyond the
framework. Operators fall into two shapes: **extension methods** that transform
an existing sequence (the vast majority), and **generator methods** on the
static `MoreEnumerable` class (`Generate`, `Sequence`, `Random`, `Unfold`) that
produce sequences from scratch.

The distinctive implementation detail is that each operator is exposed twice.
Beyond the single `MoreEnumerable` class holding every extension, the build also
emits one dedicated static class per operator — `MoreLinq.Extensions.LagExtension`,
`MoreLinq.Extensions.BatchExtension`, and so on — so callers can `using static`
a single operator without pulling in the whole namespace[^3]. These per-operator
wrapper classes are generated from [T4 templates][t4] at build time rather than
hand-written, keeping the two views in sync.

Like `System.Linq`, operators honor **deferred (lazy) execution** and stream
where they can: `Pairwise`, `Batch`, `Window*`, `Lag`/`Lead`, and `Scan` pull
from the source on demand. Others are inherently eager or buffering — `Maxima`,
`Permutations`, `Subsets`, `RandomSubset`, `Shuffle`, `PartialSort`, and the
`*Join` family must materialize part or all of the input before yielding. The
library targets multiple frameworks down to old .NET Standard levels, so it runs
on .NET Framework, .NET Core, and modern .NET alike.

## Production Notes

- **Naming collisions are the recurring footgun.** Blanket `using MoreLinq;`
  works until a BCL upgrade introduces a method with the same name and signature,
  after which overload resolution silently binds to a different implementation
  (or fails to compile). This is why the `using static` per-operator import path
  exists; on codebases that target multiple TFMs it is the safer default.
- **LINQ to Objects only.** MoreLINQ operates on in-memory `IEnumerable<T>`. It
  is *not* an `IQueryable` provider — calling these operators inside an Entity
  Framework Core query forces client-side evaluation of everything from that
  point on, which can silently pull an entire table into memory. Project to
  objects (`AsEnumerable()`/`ToList()`) deliberately before using them.
- **Buffering operators and memory.** `Permutations` and `Subsets` are
  factorial/exponential in output size; `Shuffle`, `RandomSubset`, and `Maxima`
  buffer the source. None of these are appropriate on unbounded or very large
  sequences.
- **Deprecations across majors.** `MaxBy`/`MinBy` were renamed to
  `Maxima`/`Minima` to avoid the .NET 6 BCL additions; `Concat`, `Incremental`,
  and `Windowed` were removed in 3.0/4.0 in favor of BCL or renamed
  equivalents[^4]. Upgrading a major version can surface compile errors that are
  straightforward but not automatic.
- **Enumeration side effects.** Operators like `Pipe`, `ForEach`, `Trace`, and
  `Consume` exist specifically to run side effects; combined with deferred
  execution, forgetting to enumerate (or enumerating twice) is an easy mistake.

## When to Use / When Not

**Use when:**
- You need a well-tested operator the BCL still lacks (`Batch` with buffer reuse,
  `Segment`, `Cartesian`, `FullJoin`/`LeftJoin`/`RightJoin`, `WindowLeft`,
  `RunLengthEncode`, `Scan`/`PreScan`, `Lag`/`Lead`, `Maxima`/`Minima`).
- You are on in-memory collections and want LINQ-idiomatic, lazy composition.
- You want a stable, permissively licensed dependency with a decade of use.

**Avoid when:**
- You are composing `IQueryable` for a database — the operators run client-side.
- The operator you want already exists in `System.Linq` on your target framework
  (`DistinctBy`, `MinBy`/`MaxBy`, `Chunk`, `Zip`); prefer the BCL to avoid an
  extra dependency and future collisions.
- You need only one or two helpers — copying a small operator may beat taking the
  whole package, though the source-embedding option mitigates this.

## Alternatives

- viceroypenguin/SuperLinq — actively developed superset of MoreLINQ with
  additional operators; use it when you target modern .NET only and want a
  broader, more current operator set.
- dotnet/reactive — Ix.NET's `System.Interactive` provides overlapping lazy
  operators (`Buffer`, `Scan`, `Do`, `Repeat`); use it if you already depend on
  Rx or want push/pull symmetry.
- dotnet/runtime — the BCL `System.Linq` itself; use it first for anything it now
  covers, before reaching for a third-party package.
- scottksmith95/LINQKit — for `IQueryable`/EF Core expression composition
  (`PredicateBuilder`, `AsExpandable`); a different problem than in-memory helpers.

## History

| Version | Date | Notes |
|---------|------|-------|
| Google Code origin | pre-2015 | Project started before the GitHub move[^1]. |
| 1.x | — | First widely used NuGet releases under `MoreLinq` namespace. |
| 2.0 | 2016 | Repackaged release line; `Zip` collision with .NET 4.0's own `Zip`[^3]. |
| 3.0 | 2019 | Per-operator `using static` import support; `Concat`/`Incremental` removed[^3][^4]. |
| 4.0 | ~2022 | Removed long-obsolete operators (`Concat`, `Windowed`); `MaxBy`/`MinBy` superseded by `Maxima`/`Minima`[^4]. |

## References

[^1]: MoreLINQ project home and contributor list. https://github.com/morelinq/MoreLINQ
[^2]: ".NET 6 — What's new in System.Linq" (adds `DistinctBy`, `MinBy`/`MaxBy`, `Chunk`). https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable
[^3]: MoreLINQ README, "Usage" — static-import feature and the `Zip` collision history. https://github.com/morelinq/MoreLINQ#usage
[^4]: MoreLINQ README, "Operators" — deprecation/removal notes for `Concat`, `Incremental`, `Windowed`, `MaxBy`/`MinBy`. https://github.com/morelinq/MoreLINQ#operators

## Tags

csharp, dotnet, linq, enumerable, functional, collections, lazy-evaluation, extension-methods, library, nuget
