# uber-go/mock

> Uber's maintained fork of Google's gomock — a codegen-based mocking framework wired into Go's `testing` package.

[GitHub repo](https://github.com/uber-go/mock) ·
[Package reference](https://pkg.go.dev/go.uber.org/mock) ·
[License: Apache-2.0](https://github.com/uber-go/mock/blob/main/LICENSE)

## Overview

gomock is a mocking framework for Go, split into a code generator (`mockgen`)
and a runtime library (`gomock`). You define an interface, run `mockgen` to emit
a `MockFoo` type, and in tests wire expectations against it through a fluent
`EXPECT()` DSL that asserts which methods are called, with what arguments, and
how many times — integrating with the standard `testing.T` rather than replacing
it.

This repository is a fork. The framework began as `golang/mock` inside Google,
one of the oldest mocking libraries in the ecosystem. Google stopped maintaining
it, and given heavy internal usage Uber forked it in May 2023 under the import
path `go.uber.org/mock`[^1]. The original repo is frozen; this fork is where
fixes and new modes (type-safe mocks, package mode, archive mode) have landed.

The defining tension is that gomock is strict, generated, and verbose. Mocks are
real source files checked into the repo, so they can drift out of sync with the
interface they mock, and the default `Return`/`Do` API is untyped (`...any`),
trading compile-time safety for the expectation DSL. Competing tools (moq,
mockery, testify/mock) exist largely because teams weigh that tradeoff
differently.

## Getting Started

```bash
go install go.uber.org/mock/mockgen@latest   # needs $(go env GOPATH)/bin on PATH
```

Generate a mock and use it in a test:

```go
//go:generate mockgen -source=foo.go -destination=mock_foo.go -package=foo -typed

type Foo interface {
    Bar(x int) int
}

func TestFoo(t *testing.T) {
    ctrl := gomock.NewController(t) // Finish is auto-registered via t.Cleanup
    m := NewMockFoo(ctrl)

    m.EXPECT().Bar(gomock.Eq(99)).Return(101).Times(1)

    _ = m.Bar(99) // exercised by the system under test
}
```

The opt-in `-typed` flag generates a `Return(int)` signature instead of
`Return(...any)`, restoring compile-time checking of return values.

## Architecture / How It Works

Two halves ship together. **`mockgen`** reads an interface and emits a struct
implementing it plus a recorder type behind `EXPECT()`. **`gomock`** is the
runtime: `Controller` tracks expected calls, and matchers (`gomock.Eq`, `Any`,
`Not`, `AssignableToTypeOf`, custom `Matcher` types) decide whether an actual
call satisfies an expectation.

`mockgen` has three modes, and choosing the wrong one is the most common source
of confusion[^2]:

1. **Source mode** (`-source=foo.go`) — parses the AST without compiling. Fast
   and works on non-buildable code, but cannot resolve embedded interfaces from
   other packages without `-aux_files`, and can misresolve type aliases.
2. **Package mode** (import path + symbols) — loads full type info via
   `go/packages`, resolving embeds and cross-package types correctly. Replaced
   the older reflect mode (a built-and-run reflection helper) in early fork
   releases[^3]. Requires buildable code.
3. **Archive mode** (`-archive=pkg.a`) — generates from a compiled `.a` archive,
   added later for build-system integration[^4].

The `Controller` binds to `testing.T` and registers its own `Finish()` through
`t.Cleanup`, verifying every expectation was met at test end. Ordering is
constrained with `InOrder(...)` / `.After(...)`, cardinality with `Times`,
`AnyTimes`, `MinTimes`, `MaxTimes` — all runtime state on the controller, not
reflected in the generated types.

## Production Notes

**Stale mocks are the number-one footgun.** Because generated mocks are checked
in, changing an interface silently leaves the mock on its old shape until someone
re-runs `go generate`. The standard defense is a CI step running `go generate
./...` then `git diff --exit-code`. Nothing in the tool enforces it for you.

**`ctrl.Finish()` is no longer needed** when you pass a `*testing.T` to
`NewController` — Finish registers via `t.Cleanup` automatically. Legacy code
still calls `defer ctrl.Finish()`; it is redundant, not harmful.

**Untyped by default.** Without `-typed`, `Return(...)`/`DoAndReturn(...)` accept
`any`, so a return-type mismatch surfaces as a runtime panic rather than a
compile error. Teams generally standardize on `-typed` repo-wide; migrating is
mechanical but touches every generated file.

**Matcher over-broadness.** Heavy `gomock.Any()` makes tests pass while masking
argument regressions — the mock asserts "called" but not "called correctly." A
test-design pitfall, not a bug, but a pervasive one.

**Concurrency and ordering.** When the SUT calls a mock from multiple
goroutines, `InOrder` and exact `Times(n)` expectations turn flaky unless the
test synchronizes them. The controller is safe for concurrent calls, but call
ordering cannot be enforced across nondeterministic scheduling.

**Version support.** The project follows the Go release policy — only the two
most recent Go releases are supported[^5]. Generics in mocked interfaces work
but were rough in early fork releases; keep `mockgen` current.

## When to Use / When Not

**Use when:**
- You want strict expectation-based mocks with call-count and ordering assertions.
- You already use gomock or are migrating off the unmaintained `golang/mock`.
- You want mocks driven by `go:generate` and checked in for review.
- Interfaces are stable enough that regeneration churn is acceptable.

**Avoid when:**
- You prefer hand-written or closure-based fakes with no DSL (moq, testify).
- Interfaces change constantly and stale-mock churn would dominate.
- You want config-driven bulk generation without per-interface directives (mockery).
- You dislike the untyped default and won't enforce `-typed` repo-wide.

## Alternatives

- vektra/mockery — config-driven (`.mockery.yaml`) bulk generation of
  testify-style mocks; use it when you want one config to mock everything.
- matryer/moq — minimal struct fakes backed by function fields; use it when you
  want lightweight closure-based fakes with no expectation runtime.
- stretchr/testify (mock package) — hand-written mocks with a runtime assertion
  API; use it when you want no code generation at all.
- golang/mock — the original, now unmaintained upstream; use nothing here —
  migrate to uber-go/mock, which is API-compatible.
- maxbrunsfeld/counterfeiter — generates counterfeit fakes recording invocations;
  use it when you prefer inspecting recorded calls over declaring expectations up
  front.

## History

| Version | Date | Notes |
|---------|------|-------|
| (origin) | ~2011 | Created as `golang/mock` inside Google[^1]. |
| fork | 2023-05 | Uber forks to `go.uber.org/mock`; Google repo frozen[^1]. |
| v0.4.x | 2023–2024 | `-typed` type-safe mocks; reflect mode replaced by package mode[^3]. |
| v0.5.x | 2024–2025 | Archive mode; generics and `go:generate` refinements[^4]. |

## References

[^1]: uber-go/mock README — fork rationale and origin from Google's golang/mock. https://github.com/uber-go/mock
[^2]: uber-go/mock README, "Running mockgen" — archive, source, and package modes. https://github.com/uber-go/mock#running-mockgen
[^3]: gomock package documentation. https://pkg.go.dev/go.uber.org/mock/gomock
[^4]: uber-go/mock releases. https://github.com/uber-go/mock/releases
[^5]: Go Release Policy. https://go.dev/doc/devel/release#policy

## Tags

go, golang, mocking, testing, test-doubles, code-generation, mockgen, unit-testing, interfaces, developer-tools
