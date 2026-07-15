# onsi/gomega

> Go's matcher and async-polling assertion library — the `Ω`/`Expect` DSL built for Ginkgo but usable under any test framework.

[GitHub repo](https://github.com/onsi/gomega) ·
[Official website](http://onsi.github.io/gomega/) ·
[License: MIT](https://github.com/onsi/gomega/blob/master/LICENSE)

## Overview

Gomega is the assertion half of the Ginkgo/Gomega pair, both written by Onsi Fakhouri and both originating in 2013 out of the Cloud Foundry engineering effort at Pivotal[^1]. Ginkgo provides the BDD test structure (`Describe`/`It`); Gomega provides the matchers you assert with (`Expect(x).To(Equal(y))`). The two are developed together, but Gomega has no hard dependency on Ginkgo — it works under plain `testing` via `RegisterTestingT(t)` or, preferably, a per-test `NewWithT(t)` handle. In practice most Gomega usage is inside Ginkgo suites, and the library's ergonomics are tuned for that.

Its distinguishing feature is first-class asynchronous assertion. `Eventually` and `Consistently` poll a value, function, or channel until a matcher passes (or keeps passing) within a timeout. This is why Gomega is dominant in the Kubernetes / Cloud Foundry / distributed-systems corner of Go: those suites are full of "wait until the reconciler converges" assertions that are awkward to express with a synchronous assert library.

The defining tension is verbosity and surface area. Gomega ships a large matcher catalog plus seven sub-libraries (`gstruct`, `ghttp`, `gexec`, `gbytes`, `gleak`, `gmeasure`, `gcustom`), and its DSL is more elaborate than the assert/require style most Go teams reach for by default. You adopt Gomega for the async model and the matcher richness, not for minimalism.

## Getting Started

```bash
go get github.com/onsi/gomega
```

```go
package thing_test

import (
	"testing"

	. "github.com/onsi/gomega"
)

func TestThing(t *testing.T) {
	g := NewWithT(t)               // per-test handle, parallel-safe

	g.Expect(2 + 2).To(Equal(4))
	g.Expect(err).NotTo(HaveOccurred())

	// async: poll until the matcher passes, default 1s timeout / 10ms interval
	g.Eventually(func() int {
		return queue.Len()
	}).Should(BeNumerically(">=", 3))
}
```

The dot-import (`. "github.com/onsi/gomega"`) is idiomatic here and in Ginkgo suites; it is what makes `Expect`/`Eventually`/matchers read as a DSL.

## Architecture / How It Works

A Gomega assertion is `actual` + a `GomegaMatcher`. Every matcher implements `Match(actual interface{}) (bool, error)` plus failure-message methods; `Expect` runs the matcher and, on failure, calls the registered **fail handler**. Under Ginkgo that handler records the failure and unwinds the spec; under plain `testing` (`RegisterTestingT`) it calls `t.Fatalf`. If no handler is registered, an assertion panics — a common first-run surprise.

`Eventually`/`Consistently` are the non-trivial part. They accept a value, a zero-arg function, a function returning `(value, error)`, or a function taking a `Gomega` first argument (`func(g Gomega)`) so you can make inner assertions inside the polled block. They re-invoke on an interval, compare against the matcher, and honor `.WithTimeout`, `.WithPolling`, `.WithContext`, and `.WithArguments` chained options. A returned non-nil `error` short-circuits as a failure; a function that blocks forever leaks a goroutine (see below).

The matcher catalog is broad and behavior-specific: `Equal` uses `reflect.DeepEqual`, `BeEquivalentTo` converts types before comparing, `BeIdenticalTo` uses `==`, `BeNumerically` handles operators and tolerances, `MatchJSON`/`MatchYAML` compare structurally after parsing, `ConsistOf` checks set equality regardless of order, and `HaveField` reaches into struct fields by path. `ConsistOf`'s order-independent matching is implemented with a bipartite graph matcher (`goraph`) vendored into the source tree to keep distribution self-contained[^2].

The sub-libraries are where Gomega stops being "just matchers":

- **gexec** — compile a Go binary at test time and run it, asserting on exit code and streamed output.
- **gbytes** — a thread-safe buffer with a `Say(regex)` matcher, usable with `Eventually` to assert on streaming stdout.
- **ghttp** — a configurable test HTTP server for verifying outbound client requests.
- **gstruct** — `MatchAllFields` / `MatchFields` / `MatchElements` / `PointTo` for deep, partial structural matching.
- **gleak** — a goroutine-leak detector matcher (`HaveLeaked`).
- **gmeasure** — benchmarking / experiment sampling with stats.
- **gcustom** — a builder for one-off custom matchers without writing the full interface.

## Production Notes

- **`Equal` is `reflect.DeepEqual`, which is type-strict.** `Expect(int32(1)).To(Equal(1))` fails because `1` is an `int`. Use `BeEquivalentTo` for cross-type numeric equality or `BeNumerically("==", …)` for numbers. This is the single most common Gomega footgun.
- **`RegisterTestingT` installs a process-global fail handler.** It is not safe when tests run in parallel goroutines — the handler and the `t` it points at are shared. For `t.Parallel()` or table tests, use `g := NewWithT(t)` per test and assert through `g`.
- **`Eventually` can leak goroutines.** Polling a function that spawns work, blocks internally, or never settles keeps re-running until timeout and can leave goroutines behind. Pair with `gleak` in long suites, and prefer the `.WithContext(ctx)` form so cancellation propagates.
- **Async defaults are short.** 1s timeout / 10ms polling. Integration tests that hit real I/O routinely need `.WithTimeout(30 * time.Second)`; forgetting this is a top cause of flaky "converges eventually but not in 1s" failures.
- **`Succeed()` vs `HaveOccurred()`.** Assert `Expect(doThing()).To(Succeed())` for functions returning only `error`; `NotTo(HaveOccurred())` is for a captured error value. Mixing them up produces confusing messages.
- **`gexec` needs the Go toolchain at test time** and compiles binaries into a temp dir — it is slow and unsuitable for unit-test-speed loops; scope it to integration suites.
- **Version coupling with Ginkgo.** Because they ship together, upgrading Ginkgo across major versions (notably the v1→v2 rewrite) usually means bumping Gomega too; keep them in lockstep in `go.mod`.

## When to Use / When Not

**Use when:**
- Your suite has "wait until it converges" assertions — controllers, reconcilers, message queues, streaming output. `Eventually`/`Consistently` is the reason to pick Gomega.
- You are already using Ginkgo, or want its matcher richness (`gstruct`, `ghttp`, `gexec`) without assembling equivalents.
- You value expressive, self-documenting failure messages over minimal syntax.

**Avoid when:**
- You want the smallest, most idiomatic-Go assertion surface — the dot-import DSL and large API cut against "just use `if got != want`".
- Your team already standardized on `testify` and has no async-polling need; adding Gomega alongside it splits the codebase's assertion style.
- You only need better equality diffs — `google/go-cmp` alone is lighter.

## Alternatives

- stretchr/testify — use instead when you want the mainstream Go assert/require style and have no async-polling requirement.
- google/go-cmp — use instead when the real need is readable deep-equality diffs rather than a matcher DSL; often used *with* Gomega inside a custom matcher.
- frankban/quicktest — use instead for a lighter checker-based API that still offers deep comparison and decent messages.
- matryer/is — use instead when you want near-zero-API minimalist assertions.
- onsi/ginkgo — not an alternative but the companion BDD framework; Gomega is its default matcher library.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-08 | First commit, alongside Ginkgo, at Pivotal / Cloud Foundry[^1]. |
| v1.x | ongoing | Long-lived v1 line; API stable, matchers and sub-libs added incrementally. |
| — | ~2021 | `gleak` goroutine-leak detector added. |
| — | ~2022 | `gcustom` builder, `HaveField`, and `Eventually(func(g Gomega))` polling with `.WithContext`/`.WithArguments`. |
| — | 2026-06 | Ships a Claude Code skills plugin; the repo doubles as the plugin marketplace[^3]. |

## References

[^1]: Ginkgo & Gomega originate from Onsi Fakhouri's work on Cloud Foundry at Pivotal; the Gomega repo was created 2013-08-23. https://github.com/onsi/gomega
[^2]: README — `ConsistOf` uses the embedded `goraph` (MIT) bipartite matcher to keep distribution self-contained. https://github.com/onsi/gomega#license
[^3]: README — "Using Gomega with Claude Code": `/plugin marketplace add onsi/gomega` then `/plugin install gomega@gomega`. https://github.com/onsi/gomega
[^4]: Official docs — matcher catalog and async reference. http://onsi.github.io/gomega/

## Tags

go, golang, testing, assertions, matchers, bdd, ginkgo, async-testing, test-framework, mit-license
