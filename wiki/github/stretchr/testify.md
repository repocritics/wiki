# stretchr/testify

> Assertions, mocks, and test-suite scaffolding for Go — the de facto standard test toolkit built on top of the stdlib `testing` package, not a replacement for it.

[GitHub repo](https://github.com/stretchr/testify) ·
[API docs](https://pkg.go.dev/github.com/stretchr/testify) ·
[License: MIT](https://github.com/stretchr/testify/blob/master/LICENSE)

## Overview

Testify is the most widely used third-party testing library in the Go ecosystem, with ~26k stars and a footprint in the dependency graph of a very large share of Go projects. It does not replace `go test` or the `testing` package — every testify helper takes the standard `*testing.T` as its first argument and reports through the normal test machinery. What it adds is ergonomics: readable assertions with diff-formatted failure output, a reflection-based mock framework, and an xUnit-style suite runner with setup/teardown hooks.

The defining tension is that testify is deliberately un-idiomatic Go. The canonical Go testing style is table-driven tests with hand-written `if got != want { t.Errorf(...) }` checks and no assertion library. Testify offers a more familiar assertion vocabulary to people coming from JUnit/RSpec, at the cost of a heavier API surface, reflection-driven comparisons, and a mock system that resolves method expectations by string name at runtime rather than at compile time. It is loved for cutting boilerplate and criticized for encouraging patterns the core team never intended.

The project is explicitly frozen at v1: maintainers accept no breaking changes, and a long-running discussion about a v2 has not produced one[^1]. In practice this means the API is extremely stable — code written against testify years ago still compiles — but also that known design mistakes cannot be fixed without a new major version that keeps not shipping.

## Getting Started

```bash
go get github.com/stretchr/testify
```

```go
package yours

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestSomething(t *testing.T) {
	// assert.* records a failure and CONTINUES the test
	assert.Equal(t, 123, 123, "they should be equal")

	// require.* records a failure and STOPS the test (calls t.FailNow)
	got, err := doWork()
	require.NoError(t, err)          // pointless to continue if this failed
	assert.Equal(t, "expected", got) // safe to reach only past the require
}
```

Note the argument order: `assert.Equal(t, expected, actual)` — expected first. Reversing it produces backwards "Not equal" diffs and is one of the most common mistakes the `testifylint` linter exists to catch[^2].

## Architecture / How It Works

Testify is four loosely-coupled packages sharing a comparison core:

- **`assert`** — free functions that return a `bool` (true = passed). Comparisons route through `ObjectsAreEqual`, which special-cases `[]byte` via `bytes.Equal` and otherwise falls back to `reflect.DeepEqual`. `EqualValues` additionally attempts type conversion, so `int32(1)` and `int64(1)` compare equal there but *not* under `assert.Equal` — a frequent surprise.
- **`require`** — a mirror of `assert` generated from the same source, but each function calls `t.FailNow()` on failure instead of returning. Because `FailNow` terminates the goroutine via `runtime.Goexit`, require functions must be called from the test goroutine itself; calling them inside a spawned goroutine leaks the failure and can cause race conditions[^3].
- **`mock`** — a reflection- and string-driven mock framework. You embed `mock.Mock`, register expectations with `m.On("MethodName", args...).Return(...)`, and each stub method calls `m.Called(args...)` to look up the matching expectation. Nothing is checked at compile time: method names are strings, argument matching runs through the same `ObjectsAreEqual` logic, and unmet or unexpected calls surface only at runtime (often as a panic).
- **`suite`** — a struct-based runner. Embed `suite.Suite`, define `SetupTest`/`TearDownTest`/`SetupSuite` methods, and any method named `Test*` runs as a subtest. `suite.Suite` also re-exposes the assert methods so `suite.Equal(...)` works without threading `t`.

Much of `assert`/`require` is code-generated (`go generate ./...` regenerates the `require` mirror and the `*f` formatted variants from `assert`), which is why the two packages never drift. The only real coupling between packages is the shared comparison core; you can use `assert` without ever touching `mock` or `suite`.

## Production Notes

- **The `suite` package does not support parallel tests.** Calling `t.Parallel()` inside suite methods does not behave as expected, and this has been a known open limitation for years (#934)[^4]. If you rely on parallelism, use plain `Test*` functions or `t.Run` subtests, not `suite`.
- **Mocks fail silently if you forget `AssertExpectations`.** Registering `.On(...)` expectations does nothing on its own — you must call `mockObj.AssertExpectations(t)` (typically in a `t.Cleanup`) or unmet expectations never fail the test. Conversely, an *unexpected* call with no matching `.On` panics mid-test.
- **`assert.Equal` is strict about types.** `assert.Equal(t, 1, int64(1))` fails because the dynamic types differ. Reach for `EqualValues` for cross-type numeric comparisons, or `InDelta`/`InEpsilon` for floats. Comparing large structs dumps a full `go-spew` diff that can be enormous.
- **Typed-nil handling.** `assert.Nil` and `assert.Error` use reflection to detect nil-through-interface, so an interface holding a `(*T)(nil)` is correctly reported as nil — behavior that differs from a naked `== nil` and occasionally surprises people debugging error checks.
- **Dependency weight.** Testify pulls in `davecgh/go-spew`, `pmezard/go-difflib`, `stretchr/objx` (for `mock`), and `gopkg.in/yaml.v3`. The transitive `yaml.v3` in particular is a recurring complaint for projects that want a minimal test dependency and don't use `assert.YAMLEq`.
- **Lint it.** Because so many footguns (wrong arg order, `assert` where `require` is needed, `Equal` vs `ErrorIs`) are mechanical, running `testifylint` under `golangci-lint` catches most of them and is close to mandatory on serious codebases[^2].

## When to Use / When Not

**Use when:**
- You want readable assertions and diff output without hand-writing comparison boilerplate.
- You need mocks for interfaces and don't want to hand-roll fakes, or you pair it with a generator like `mockery`.
- Your team comes from xUnit-style frameworks and values `suite` setup/teardown structure.

**Avoid when:**
- You want maximally idiomatic Go: the core team and many library authors prefer table-driven tests with plain `if`/`t.Errorf` and no assertion dependency.
- You need compile-time-safe mocks: prefer generated mocks against interfaces (`mockery`, or Uber's `gomock`) over testify's string-keyed `mock`.
- You need parallel suite tests, or you want to keep test dependencies minimal.

## Alternatives

- matryer/is — tiny, zero-dependency assertion helper; use instead of testify's `assert` when you want minimalism.
- uber-go/mock (formerly golang/mock) — code-generated, interface-typed mocks; use instead of testify `mock` when you want compile-time safety.
- vektra/mockery — generates testify-compatible mocks from interfaces; use alongside testify to remove hand-written mock boilerplate.
- google/go-cmp — `cmp.Diff` for deep equality with custom options; use instead of `assert.Equal` when you need fine-grained comparison control.
- onsi/ginkgo + onsi/gomega — full BDD framework and matcher library; use instead of `suite` when you want an expressive spec-style runner.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2012-10-16 | Repo created under stretchr[^5]. |
| v1.1.4 | 2016-09-25 | Early tagged v1 line. |
| v1.3.0 | 2019-01-05 | |
| v1.6.0 | 2020-05-29 | |
| v1.7.0 | 2021-01-13 | |
| v1.9.0 | 2024-03-01 | |
| v1.10.0 | 2024-11-23 | |
| v1.11.1 | 2025-08-27 | Latest patch; still no v2, project frozen at v1[^1]. |

## References

[^1]: "Testify is being maintained at v1, no breaking changes will be accepted." README note and v2 discussion. https://github.com/stretchr/testify/discussions/1560
[^2]: testifylint — linter for common testify mistakes, integrated into golangci-lint. https://github.com/Antonboom/testifylint
[^3]: `require` package docs — must be called from the test goroutine; see `testing.T.FailNow`. https://pkg.go.dev/github.com/stretchr/testify/require
[^4]: "suite package does not support parallel tests" — open issue #934. https://github.com/stretchr/testify/issues/934
[^5]: Repository metadata (created 2012-10-16), GitHub API. https://github.com/stretchr/testify

## Tags

go, golang, testing, assertions, mocking, test-suite, unit-testing, stdlib-compatible, developer-tools, code-generation
