# uber-go/multierr

> Combine several Go `error` values into one, with helpers built for the `defer`-and-close pattern.

[GitHub repo](https://github.com/uber-go/multierr) ·
[Official docs](https://go.uber.org/multierr) ·
[License: MIT](https://github.com/uber-go/multierr/blob/master/LICENSE.txt)

## Overview

`multierr` is a small, dependency-light library from Uber's Go group (the same
family as `zap` and `fx`, both of which use it internally) for aggregating
multiple `error` values into a single `error`[^1]. It was created in 2017, at a
time when the standard library gave you no idiomatic way to say "these three
things failed" — you either returned the first error and dropped the rest, or
hand-rolled a slice-of-errors type in every package.

The defining fact about `multierr` in 2026 is that Go's own standard library
caught up. Go 1.20 (February 2023) added `errors.Join`, which does the core job
of `Combine`: bundle N errors, filter out the nils, and expose them to
`errors.Is` / `errors.As` via `Unwrap() []error`[^2]. So the honest framing is:
`multierr` is a mature, stable, still-widely-imported library whose *basic* use
case is now covered by the stdlib, but which retains a meaningfully richer API —
`Errors()` extraction, in-place append optimization, and first-class `defer`
helpers (`AppendInto`, `AppendInvoke`, `Close`) that `errors.Join` does not
offer. The README pins the project as stable with no breaking changes before a
hypothetical 2.0[^1]; development is effectively feature-complete rather than
abandoned.

## Getting Started

```bash
go get -u go.uber.org/multierr@latest
```

```go
package main

import (
	"fmt"

	"go.uber.org/multierr"
)

func main() {
	err := multierr.Combine(
		validateName(),   // may be nil
		validateEmail(),  // may be nil
		validateAge(),    // may be nil
	)
	// nil inputs are dropped; if all are nil, err == nil.
	fmt.Println(err)                 // "bad name; bad age"  (single line)
	fmt.Printf("%+v\n", err)         // multi-line bulleted form

	for _, e := range multierr.Errors(err) {
		fmt.Println("-", e)          // iterate the individual errors
	}
}
```

The `defer` idiom that motivates the library — accumulate a `Close` error
without clobbering the real return value:

```go
func process(name string) (err error) {
	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer func() { err = multierr.Append(err, f.Close()) }()
	// ... do work; a failure here is preserved alongside any Close error ...
	return nil
}
```

## Architecture / How It Works

Internally a combined error is a `*multiError` wrapping a `[]error` slice. Its
`Error()` method joins children with `"; "` for the single-line form; the
`fmt.Formatter` implementation produces a `the following errors occurred:`
bulleted block under `%+v`. It also exposes `Errors() []error` for programmatic
extraction and, on Go 1.20+, `Unwrap() []error` so standard-library
`errors.Is` and `errors.As` traverse every combined error[^2].

The API surface is deliberately narrow:

- **`Combine(...error) error`** — variadic aggregate; nils are filtered, and a
  passed-in `*multiError` is flattened rather than nested.
- **`Append(left, right error) error`** — the two-argument workhorse used in
  `defer`. It has an in-place optimization: when `left` is already a
  `*multiError` with spare slice capacity, `right` is appended without
  allocating a new backing array. This is what makes appending in a loop cheap.
- **`AppendInto(*error, error) bool`** — appends into a pointer target and
  returns whether the appended error was non-nil; convenient for loop bodies.
- **`AppendInvoke(*error, Invoker)`** with the `Close(io.Closer)` and
  `Invoke(func() error)` helpers — lets you write the close-on-defer pattern as
  a single one-line `defer` instead of a `defer func(){...}()` closure.

Because the underlying type is unexported, callers only ever deal in `error`
values — you cannot type-assert to a concrete `multierr` type, only call
`multierr.Errors()`. That opacity is intentional and is the "idiomatic" claim in
the README.

## Production Notes

- **`Append`'s in-place mutation is not concurrency-safe.** The optimization
  that makes loop-appends fast mutates the left operand's backing slice when it
  has capacity. Two goroutines appending into the same accumulated error value
  can race. If you aggregate errors across goroutines, guard the shared `error`
  with a mutex, or have each goroutine build its own and `Combine` at the join
  point.
- **`Combine` / `Append` flatten, and order is preserved.** Nesting a combined
  error inside another does not create a tree; children are spliced in. Do not
  rely on grouping structure surviving a round-trip.
- **`Errors()` normalizes single errors.** `multierr.Errors(err)` returns `nil`
  for a nil input, a one-element slice for an ordinary (non-multierr) error, and
  the full list for a combined one. Code that iterates the result works
  uniformly regardless of how many errors were actually combined.
- **String format is stable but not machine-parseable.** The `"; "` separator is
  fine for logs; do not parse it back into constituent errors — use `Errors()`.
- **Interop depends on your Go version.** Full `errors.Is`/`errors.As`
  traversal over all children relies on `Unwrap() []error`, which is only
  meaningful on Go 1.20+. On older toolchains behavior around multi-error
  traversal differs; upgrading the toolchain is the fix.

## When to Use / When Not

**Use when:**
- You accumulate a `Close()` / cleanup error in `defer` without losing the main
  return value — the `AppendInto` / `AppendInvoke` / `Close` helpers exist
  precisely for this and have no stdlib equivalent.
- You need to iterate the individual errors after the fact via `Errors()`.
- You're already in the uber-go ecosystem (`zap`, `fx`) and want a consistent,
  zero-surprise dependency.
- You append errors in a hot loop and want the in-place slice optimization.

**Avoid when:**
- You're on Go 1.20+, only need to bundle a fixed set of errors, and want zero
  dependencies — `errors.Join` from the standard library is enough.
- You need rich structured errors: stack traces, error codes, wire encoding, or
  redactable metadata. `multierr` deliberately does none of that.
- You want a tree of grouped errors preserved through combination — it flattens.

## Alternatives

- golang/go `errors.Join` (stdlib, Go 1.20+) — use instead when you only need to
  bundle errors and are on a modern toolchain; no third-party dependency.
- hashicorp/go-multierror — use instead when you want configurable formatting
  and an explicit exported `*Error` type you can inspect directly.
- cockroachdb/errors — use instead when you need stack traces, error codes, and
  network-safe encoding of errors across process boundaries.
- pkg/errors — use instead (with caution; it is archived) only for wrapping a
  single error with a stack trace, a different job than aggregation.

## History

| Version | Date (approx.) | Notes |
|---------|------|-------|
| 1.0.0 | 2017 | First stable tag; API frozen until a hypothetical 2.0[^1]. |
| 1.7.0 | 2021 | `AppendInvoke` plus `Close` / `Invoke` helpers for `defer`. |
| — | 2023-02 | Ecosystem context: Go 1.20 ships `errors.Join`, overlapping the core use case[^2]. |
| 1.11.0 | 2023 | Latest tagged release as of the repo's last push (2024-04). |

## References

[^1]: multierr README — features, stability statement, MIT license. https://github.com/uber-go/multierr/blob/master/README.md
[^2]: Go 1.20 release notes — `errors.Join` and the `Unwrap() []error` convention. https://go.dev/doc/go1.20#errors

## Tags

go, golang, error-handling, errors, multierror, aggregation, defer, uber-go, stdlib-interop, library
