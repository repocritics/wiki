# google/go-cmp

> A safer, more configurable replacement for `reflect.DeepEqual`, built for equality checks in Go tests.

[GitHub repo](https://github.com/google/go-cmp) ·
[Docs](https://pkg.go.dev/github.com/google/go-cmp/cmp) ·
[License: BSD-3-Clause](https://github.com/google/go-cmp/blob/master/LICENSE)

## Overview

`go-cmp` (import path `github.com/google/go-cmp/cmp`) is a comparison library from Google, first tagged in 2017[^1]. It exists because `reflect.DeepEqual` — the standard library's structural equality check — has two failure modes that hurt tests: it silently compares unexported fields (coupling tests to internal representation), and when values differ it tells you only `true`/`false`, not *what* differed. `cmp.Equal` addresses the first by refusing to touch unexported fields unless you explicitly opt in, and `cmp.Diff` addresses the second by rendering a human-readable, structured diff of the two values.

The library is deliberately scoped to tests. Its own documentation states that it is intended for use in tests, that performance is not a design goal, and that its output format may change between releases[^2]. This is the defining tension of the project: it is one of the most widely used packages in the Go testing ecosystem, yet the maintainers actively discourage depending on it in production code paths or asserting against the exact text of its diff output. Treating `cmp.Diff` output as a stable API — for example, snapshotting it into golden files that must match byte-for-byte — is a known way to get broken by a patch release.

The other defining trait is that equality is *configurable but explicit*. Out of the box `cmp` panics rather than guesses when it hits something ambiguous (most commonly an unexported field). You resolve the panic by passing an `Option` that states your intent. This trades convenience for a test suite that fails loudly when types change shape, rather than passing or failing for reasons the author did not intend.

## Getting Started

```
go get github.com/google/go-cmp/cmp
```

The canonical idiom is `cmp.Diff` inside a table-driven test, printed on mismatch:

```go
import (
    "testing"

    "github.com/google/go-cmp/cmp"
)

func TestBuildUser(t *testing.T) {
    got := BuildUser("ada")
    want := User{Name: "ada", Roles: []string{"admin"}}

    // Diff returns "" when equal. Convention: (-want +got).
    if diff := cmp.Diff(want, got); diff != "" {
        t.Errorf("BuildUser() mismatch (-want +got):\n%s", diff)
    }
}
```

For a plain boolean, use `cmp.Equal(want, got, opts...)`. Both take the same variadic `Option` list.

## Architecture / How It Works

`cmp` walks two values in lockstep using reflection, descending recursively. At each node it decides how to compare using a fixed precedence:

1. **Options** you supplied (`cmp.Comparer`, `cmp.Transformer`, `cmp.FilterField`, and the `cmpopts` helpers) that match the current path.
2. An **`Equal` method** on the type — any `func (T) Equal(T) bool` (or `Equal(I) bool` for an interface argument) is used automatically. This lets a package author define equality once and have every test respect it.
3. Otherwise, **structural recursion** over the kind (struct fields, slice elements, map entries, pointers), bottoming out at primitive comparison.

Two design decisions shape everything downstream. First, **unexported struct fields are off-limits by default**: reaching one without permission is a panic, not a silent read. You grant access narrowly with `cmpopts.IgnoreUnexported(T{})` or `cmp.AllowUnexported(T{})`, or better, by giving the type an `Equal` method. Second, `Option`s are composed and matched by *path* — `cmp` tracks the exact route from the root to the current node (`Field`, `Index`, `MapIndex`, `Transform`), so options like `cmpopts.IgnoreFields(T{}, "UpdatedAt")` apply surgically rather than globally.

The two workhorse transforms most teams reach for are `cmp.Comparer` (replace equality for a type — e.g. `cmpopts.EquateApprox` for floats within a tolerance, or `cmp.Comparer(bytes.Equal)` for byte slices) and `cmp.Transformer` (map a value to another value before comparing — e.g. sort a slice whose order is irrelevant via `cmpopts.SortSlices`). The `cmpopts` subpackage is a curated set of these for common cases: `EquateEmpty` (treat nil and zero-length maps/slices as equal), `IgnoreFields`, `IgnoreUnexported`, `EquateApprox`, `SortSlices`, `SortMaps`, and `EquateComparable` (added in v0.6.0)[^3].

The diff reporter is a separate concern layered on top: it records where the walk found inequalities and formats them with heuristics tuned for signal-to-noise — batching primitive lists onto single lines, using triple-quoted blocks for multi-line strings, and truncating very large structures. The v0.5.0 release was largely a rewrite of this reporter[^4].

## Production Notes

- **It is a test-time tool.** The maintainers explicitly say the diff format is not stable and may change[^2]. Do not assert on the literal text of `cmp.Diff`, and do not use `cmp` on a hot path — it is reflection-heavy and makes no performance guarantees. For runtime equality, hand-write it or use an `Equal` method directly.
- **Panics are the common first encounter.** `cmp.Equal(a, b)` on structs with unexported fields (very common — `time.Time`, most third-party structs) panics with a message pointing at the offending field. This is by design, but it surprises newcomers who expect `reflect.DeepEqual` semantics. The fix is an option, not a flag: `cmpopts.IgnoreUnexported(...)`, an `Equal` method, or `protocmp.Transform()` for protobuf messages.
- **Protocol buffers need a transform.** Comparing `proto.Message` values directly will panic on internal state fields. Use `protocmp.Transform()` from `google.golang.org/protobuf/testing/protocmp`; v0.7.0 improved the panic message to detect proto types and point you there[^3].
- **`EquateEmpty` is almost always wanted.** Without it, a function returning `nil` and one returning `[]T{}` compare unequal — a frequent source of confusing red tests. Many teams add it to a shared default option set.
- **Argument order is a convention, not enforced.** `cmp.Diff` does not know which side is "want"; the `(-want +got)` labeling only matches if you pass `want` first. Getting this backwards produces diffs that read inverted.
- **Zero third-party dependencies.** The module pulls in nothing outside the standard library, which keeps it safe to add to any test suite. v0.7.0 (2025) raised the minimum Go version; pin an older tag if you must build with an old toolchain[^3].

## When to Use / When Not

**Use when:**
- You are comparing structured values in Go tests and want a readable diff on failure.
- You need equality that ignores irrelevant fields (timestamps, IDs) or tolerates float imprecision or slice ordering.
- You want tests to fail loudly when a type gains an unexported field, rather than silently changing behavior.

**Avoid when:**
- You need equality on a performance-sensitive runtime path — this is a test tool by design.
- You want a full assertion framework with fluent matchers and failure messages; `cmp` only computes equality/diffs, it does not fail the test for you.
- You are tempted to snapshot the diff text into golden files — the format is explicitly unstable.

## Alternatives

- stretchr/testify — use when you want a full `assert`/`require` matcher suite; its `ObjectsAreEqual` is less precise about unexported fields but ergonomically it fails the test for you.
- go-test/deep — use when you want a lightweight `deep.Equal` returning a `[]string` of differences with no options system.
- r3labs/diff — use when you need a programmatic changelog of differences (for audit/sync logic), not a test-failure string.
- gotestyourself/gotest.tools — use when you want golden-file support and helpers; its `assert.DeepEqual` wraps `go-cmp` and integrates with `*testing.T`.
- reflect.DeepEqual (stdlib) — use when you want no dependency and do not care about unexported-field safety or readable output.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2017-08-09 | Initial tagged release[^1]. |
| v0.2.0 | 2018-02-20 | Early API stabilization. |
| v0.3.0 | 2019-04-29 | Continued option/reporter work. |
| v0.4.0 | 2020-01-07 | Pre-reporter-rewrite line. |
| v0.5.0 | 2020-06-18 | Major diff-reporter rewrite for signal-to-noise[^4]. |
| v0.5.9 | 2022-09-08 | Last of the long v0.5.x maintenance line. |
| v0.6.0 | 2023-10-10 | Added `cmpopts.EquateComparable`; removed purego fallbacks[^3]. |
| v0.7.0 | 2025-02-21 | Compare-func support in `SortSlices`/`SortMaps`; better proto panic messages[^3]. |

Cadence is slow and deliberate: as of mid-2026 the package sits at ~4.7k stars with the last release in early 2025, characteristic of a mature, near-feature-complete utility rather than an inactive one — commits continue to land on `master`[^5].

## References

[^1]: go-cmp release tags. https://github.com/google/go-cmp/tags
[^2]: Package documentation, `cmp` — usage and stability notes ("intended to only be used in tests", output may change). https://pkg.go.dev/github.com/google/go-cmp/cmp
[^3]: go-cmp releases (v0.6.0, v0.7.0 notes). https://github.com/google/go-cmp/releases
[^4]: go-cmp v0.5.0 release notes — reporter rewrite. https://github.com/google/go-cmp/releases/tag/v0.5.0
[^5]: Repository activity, `master` branch. https://github.com/google/go-cmp

## Tags

go, testing, equality, diff, reflection, assertions, test-tooling, comparison, golang, library
