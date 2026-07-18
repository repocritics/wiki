# dlang/phobos

> The standard library of the D programming language — range-based, template-heavy, and permanently bundled with the compiler.

[GitHub repo](https://github.com/dlang/phobos) ·
[Official website](https://dlang.org) ·
[License: BSL-1.0](https://github.com/dlang/phobos/blob/master/LICENSE_1_0.txt)

## Overview

Phobos is the standard library that ships with every D compiler release[^1]. The name comes from the larger moon of Mars — a nod to D's origin at Walter Bright's Digital Mars (the companion repo of C bindings is named Deimos, after the smaller moon). It covers the usual standard-library ground — algorithms, containers, regex, Unicode, parallelism, message-passing concurrency, math — plus D-specific territory: `std.traits` and `std.meta` for compile-time reflection, and a regex engine (`ctRegex`) that can compile patterns at compile time via CTFE.

Its design identity is the **range abstraction**, introduced in Andrei Alexandrescu's 2009 "On Iteration" essay[^2] and realized in `std.algorithm`/`std.range`: lazy, composable adaptors chained with UFCS (`data.filter!(...).map!(...).take(5)`), specialized at compile time via template constraints. This predates Rust iterators and C++20 ranges and remains one of the more coherent generic-programming stdlib designs. The tension is that Phobos is also a two-decade accumulation: it leans on D's garbage collector throughout, carries known design mistakes (string auto-decoding, below) that were judged too breaking to remove, and cannot version independently of the compiler.

Raw GitHub numbers need interpretation. 1,253 stars is small because D community attention concentrates on the dlang/dmd compiler repo; Phobos itself is actively maintained (pushed within the last day as of mid-2026) with a backlog of ~1,000 open issues accumulated over its long bug-tracker history. This is a living project in a niche language, not an abandoned one.

## Getting Started

Phobos is not installed separately — it comes with the compiler (DMD, LDC, or GDC):

```bash
curl -fsS https://dlang.org/install.sh | bash -s dmd
# macOS alternative: brew install dmd   (or ldc)
```

```d
// squares.d — run with: rdmd squares.d
import std.stdio, std.algorithm, std.range;

void main()
{
    iota(1, 1_000)
        .filter!(n => n % 3 == 0)
        .map!(n => n * n)
        .take(5)
        .writeln;   // [9, 36, 81, 144, 225] — the chain is lazy; nothing
}                   // is computed until writeln consumes it
```

## Architecture / How It Works

**Two-layer split.** Below Phobos sits **druntime** (`core.*`): the GC, thread support, associative arrays, exception machinery, and TypeInfo. Phobos (`std.*`) builds on it. druntime was folded into the dlang/dmd repository in 2022[^3], so this repo now contains only the `std` namespace; the runtime it depends on is versioned with the compiler.

**Ranges, not iterators.** The core primitives are `empty` / `front` / `popFront`, with capability tiers (input, forward, bidirectional, random-access) detected by template constraints like `isInputRange!R`. Algorithms are templates instantiated per concrete range type — zero-overhead in principle, but every chain link is a distinct template instantiation, which shows up in compile times and symbol bloat.

**Compile-time execution everywhere.** CTFE and templates are used aggressively: `std.regex.ctRegex` generates a specialized matcher at compile time, `std.format` parses format strings statically, and `std.traits`/`std.meta` implement reflection as ordinary library code. Attributes (`@safe`, `pure`, `nothrow`, `@nogc`) are inferred for template code, so a Phobos chain is only as `@nogc` as its weakest link.

**GC coupling.** Exceptions, array concatenation, closures, and many convenience APIs allocate through druntime's stop-the-world, non-moving, partially conservative collector. A `@nogc` subset of Phobos exists but is patchy — this is the single biggest architectural constraint and the reason the Mir ecosystem exists alongside it.

**Auto-decoding.** Phobos's oldest self-acknowledged mistake: `string` (an array of UTF-8 code units) is presented to range algorithms as a range of `dchar` code points, silently decoding on every step. It costs performance, still doesn't give grapheme-level correctness, and breaks random access. Removal has been debated for a decade and rejected as too breaking; the escape hatches are `.byCodeUnit` and `.representation`[^4].

**Compiler lockstep.** Phobos has no independent version. Its version *is* the compiler front-end version (2.x); LDC and GDC each ship a matched port. You cannot upgrade the stdlib without upgrading the compiler.

## Production Notes

- **Version lock is the operational reality.** A stdlib bug fix means a compiler upgrade. GDC tracks GCC's release cadence and can lag the DMD front-end by a year or more — code targeting GDC must target older Phobos.
- **GC pauses are real but manageable.** The collector only runs on allocation, so the standard mitigations are: preallocate, use `@nogc` hot paths, call `GC.disable` around latency-critical sections, and profile with `--DRT-gcopt=profile:1`. Fully GC-free D (`-betterC`) excludes almost all of Phobos, not just the allocating parts — exceptions and TypeInfo go away with druntime.
- **Auto-decoding will find you.** `walkLength` on a `string` is O(n) decoding, `retro` on one surprises people, and algorithm results change type (`dchar` vs `char`). Wrap hot string code in `.byCodeUnit` or `.representation` before benchmarking anything else.
- **Template bloat.** Deep range chains inflate compile times, binary size, and — worst — error messages, which report failed template constraints rather than the actual mismatch. `-verrors=context` and instantiating chains incrementally are the debugging techniques.
- **`std.experimental` is a limbo, not a pipeline.** `std.experimental.allocator` has sat there since the mid-2010s without graduating. Treat anything under that namespace as unstable API you may have to vendor.
- **Removals happen.** `std.stream` is long gone and `std.xml` was deprecated in favor of third-party dxml; the dlang/undeaD repo hosts resurrected corpses if you're migrating legacy code[^5]. Deprecation cycles are announced in release changelogs[^6] and typically span many releases, but they do complete.
- **`std.net.curl` requires libcurl at runtime** (loaded dynamically) — a deployment surprise on minimal containers.
- **A ground-up redesign ("Phobos v3") is in progress** in the D community, aiming to shed auto-decoding and other legacy decisions. It is design/early-implementation stage — do not plan production work around it yet.

## When to Use / When Not

**Use when:**
- You're writing D — Phobos is the default, and its range idioms are the language's lingua franca.
- Batch tools, code generation, and script-like programs (`rdmd`) where GC pauses are irrelevant and lazy range pipelines shine.
- You want compile-time metaprogramming (reflection, CTFE regex, static introspection) with stdlib support rather than as a bolt-on.

**Avoid when:**
- You need `-betterC` or `@nogc`-first code — most of Phobos is off the table; use Mir instead.
- Hard real-time or allocation-sensitive services where a stop-the-world GC in the stdlib's hot paths is disqualifying.
- You need async I/O — Phobos has fibers and message passing but no async networking story; that lives in vibe.d.
- You need a stdlib that upgrades independently of the compiler toolchain.

## Alternatives

- libmir/mir-algorithm — use instead when you need `@nogc`/`-betterC`-compatible algorithms and ndslice numerics; the de facto Phobos replacement for high-performance D.
- vibe-d/vibe.d — use alongside (not instead) when you need async I/O, HTTP, and an event loop, none of which Phobos provides.
- dlang/undeaD — use when migrating legacy code that depended on modules removed from Phobos (`std.stream`, `std.xml`).
- ziglang/zig — use instead when the GC is the dealbreaker and you want a comptime-heavy stdlib in a different systems language.
- rust-lang/rust — use instead when you want lazy iterator pipelines with compile-time memory safety and no GC.

## History

| Version | Date | Notes |
|---------|------|-------|
| D 1.0 | 2007-01 | First stable D; minimal Phobos v1. Community dissatisfaction spawned Tango, a rival stdlib — the "Phobos/Tango split" that fragmented D1's ecosystem[^7]. |
| D 2.000 | 2007-06 | D2 begins; `const`/`immutable` groundwork enables the Phobos redesign. |
| — | 2009–2010 | Range-based `std.algorithm`/`std.range` land (Alexandrescu)[^2]; *The D Programming Language* published 2010. |
| D1 EOL | 2012-12 | D1 discontinued; the Tango schism formally over — D2 Phobos is the sole stdlib. |
| 2.100 | 2022-05 | Round-number milestone of the ~6-releases/year train[^6]. |
| — | 2022 | druntime merged into dlang/dmd; this repo becomes `std.*` only[^3]. |
| — | 2024– | Phobos v3 redesign effort underway. |

## References

[^1]: Phobos API documentation. https://dlang.org/phobos/
[^2]: Andrei Alexandrescu, "On Iteration" — 2009. https://www.informit.com/articles/article.aspx?p=1407357
[^3]: dlang/dmd repository (contains the compiler frontend and druntime since the 2022 merge). https://github.com/dlang/dmd
[^4]: `std.utf.byCodeUnit` — the auto-decoding escape hatch. https://dlang.org/phobos/std_utf.html#byCodeUnit
[^5]: dlang/undeaD — modules removed from Phobos, kept on life support. https://github.com/dlang/undeaD
[^6]: D changelog (per-release Phobos changes and deprecations). https://dlang.org/changelog/
[^7]: Wikipedia, "D (programming language)" — history of the D1 Phobos/Tango standard-library split. https://en.wikipedia.org/wiki/D_(programming_language)

## Tags

d, dlang, standard-library, ranges, templates, metaprogramming, compile-time, generic-programming, garbage-collection, boost-license
