# onsi/ginkgo

> A BDD-style testing framework for Go that builds a spec tree on top of the standard `testing` package.

[GitHub repo](https://github.com/onsi/ginkgo) ·
[Official website](https://onsi.github.io/ginkgo/) ·
[License: MIT](https://github.com/onsi/ginkgo/blob/master/LICENSE)

## Overview

Ginkgo is a testing framework for Go authored by Onsi Fakhouri, first developed in 2013 at Pivotal to test Cloud Foundry components[^1]. It provides a Behavior-Driven Development (BDD) domain-specific language — `Describe`, `Context`, `When`, `It`, `BeforeEach` — familiar to anyone coming from RSpec, Jasmine, or Mocha. It is almost always paired with Gomega (onsi/gomega), a separate matcher library from the same author: Ginkgo supplies the runner and the spec tree, Gomega supplies the assertions.

The defining tension is cultural. Go's community strongly favors minimal, table-driven tests using the standard `testing` package, and Ginkgo deliberately goes the other way with a nested-closure DSL and a large CLI. In exchange you get first-class spec randomization, process-level parallelism, structured setup/teardown, labels, and machine-readable reporting out of the box. Whether that trade is worth it is a perennial debate in Go shops; Ginkgo tends to earn its keep on large integration and end-to-end suites — Kubernetes' e2e tests are the canonical large adopter[^4] — and to feel like overhead on small unit packages.

Version 2, released in 2022, was a substantial rewrite[^2]: a formal two-phase tree model, interruptible nodes via `context.Context`, `Ordered`/`Serial` containers, decorator-based configuration, and the removal of custom reporters and `Measure`. Migrating a v1 suite is non-trivial, which is the main reason older codebases still pin v1.

## Getting Started

```bash
go install github.com/onsi/ginkgo/v2/ginkgo@latest   # the CLI
go get github.com/onsi/ginkgo/v2                      # the DSL
go get github.com/onsi/gomega/...                     # matchers
```

Bootstrap a suite inside the package you want to test:

```bash
cd path/to/books
ginkgo bootstrap    # writes books_suite_test.go
ginkgo generate     # writes books_test.go
```

```go
package books_test

import (
	"testing"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
)

// The single Go test that hosts the whole Ginkgo suite.
func TestBooks(t *testing.T) {
	RegisterFailHandler(Fail)   // wire Gomega assertion failures into Ginkgo
	RunSpecs(t, "Books Suite")
}

var _ = Describe("Book", func() {
	var book *Book

	BeforeEach(func() {
		book = &Book{Title: "Les Misérables", Pages: 1488}
	})

	It("categorizes long books", func() {
		Expect(book.Category()).To(Equal(CategoryLong))
	})
})
```

```bash
ginkgo -r -p --randomize-all   # recurse, parallel, randomize every spec
```

## Architecture / How It Works

Ginkgo runs in **two distinct phases**, and understanding the split explains most of its behavior:

1. **Tree Construction.** When the suite loads, every container closure (`Describe`, `Context`, `When`) executes exactly once to build a tree of nodes. Setup and subject closures are *registered* here, not run.
2. **Run Phase.** Ginkgo walks the leaf `It` nodes. For each one it executes the chain of `BeforeEach`/`JustBeforeEach` setup closures from root to leaf, then the `It` body, then `AfterEach` closures unwinding back up.

A "spec" is therefore one `It` leaf plus its inherited setup chain. `RunSpecs` is the bridge into Go's own test runner: a single `func TestXxx(t *testing.T)` hosts the entire suite, so to `go test` the whole thing looks like one test.

Other internals:

- **Randomization.** Top-level containers are shuffled by default; `--randomize-all` shuffles every spec. A printed `--seed` makes any run reproducible, which is how Ginkgo surfaces order-dependent tests.
- **Parallelism.** `ginkgo -p` compiles the suite (`go test -c`) and forks it into N *operating-system processes*, not goroutines[^3]. A coordinating server distributes specs and serializes output. Because processes share no memory, `SynchronizedBeforeSuite`/`SynchronizedAfterSuite` run one-time setup on a single process and hand results to the rest, and `GinkgoParallelProcess()` returns a per-process index for partitioning ports, databases, and fixtures.
- **Decorators.** Node behavior is configured by values passed alongside the closure: `Ordered`, `Serial`, `Label(...)`, `FlakeAttempts(n)`, `MustPassRepeatedly(n)`, `SpecTimeout`, `NodeTimeout`, `GracePeriod`.
- **Interruptibility.** Nodes that accept a `SpecContext` receive a context canceled on timeout or Ctrl-C, enabling graceful cleanup. `DeferCleanup` registers teardown inline without a matching `AfterEach`.

## Production Notes

**The closure-variable footgun.** The single most common bug: assigning a value at container scope instead of inside `BeforeEach`. Because container bodies run once at tree-construction time, such a value is shared across every spec and never reset — mutations bleed between specs, and randomization makes the failure non-deterministic. The rule is: *declare* variables in the container, *assign* them in `BeforeEach`.

**Two ways to run, subtly different.** `go test ./...` runs Ginkgo specs in-process and serially — fine for CI that only cares about pass/fail, but you lose parallelism, `ginkgo watch`, aggregated progress, and CLI filtering. The `ginkgo` binary is required for `-p` parallelism and most of the advertised tooling. Teams frequently trip over specs behaving differently under the two invocations.

**Programmatic focus fails CI on purpose.** Committing an `FIt`/`FDescribe` (or a `Focus` decorator) makes Ginkgo exit non-zero even though the focused specs pass, so a stray focus can't silently disable the rest of your suite. Surprising the first time, but intentional; controlled by `--fail-on-focus`.

**Parallel state is your problem.** Since parallel specs are separate processes hitting shared external resources (databases, ports, temp dirs), you must partition per `GinkgoParallelProcess()` or serialize with `Serial`. Suites that assumed goroutine-style shared memory break when parallelized.

**v1 → v2 migration is real work.** V2 removed custom reporters, changed `BeforeSuite` semantics, moved benchmarking out to `gmeasure`, and deprecated `GinkgoT()` patterns. The official migration guide is thorough[^2] but large suites still take real effort, which is why v1 lingers.

**Idiomatic-Go friction.** Coverage attribution, IDE test discovery, and `go test -run` targeting all work less cleanly against the nested DSL than against table-driven tests. Editor integrations that expect one Go function per test see a single `TestXxx`. Budget for tooling papercuts.

## When to Use / When Not

**Use when:**
- You have large integration or end-to-end suites where structured setup, process-level parallelism, and labels/filters pay off.
- You want spec randomization and reproducible seeds to catch order dependence.
- Your team already thinks in BDD (`Describe`/`Context`/`It`) and values readable spec narratives over minimalism.
- You need built-in JUnit/JSON reporting for CI dashboards without extra glue.

**Avoid when:**
- You're writing small unit packages — idiomatic table-driven tests with the standard library are simpler and better supported.
- Your team values Go's minimalism and wants tooling (coverage, IDE, `-run`) to behave conventionally.
- You don't want a second binary and a DSL in the test dependency graph.
- You're allergic to a heavy test framework in an otherwise dependency-light codebase.

## Alternatives

- stretchr/testify — the most-used Go test helper; assertions plus an optional suite type. Use when you want idiomatic table-driven tests with less DSL.
- Go standard library `testing` — subtests via `t.Run` and table cases. Use when you want zero framework and maximum tooling compatibility.
- cucumber/godog — Gherkin/Cucumber BDD for Go. Use when specs must be readable and editable by non-engineers.
- smartystreets/goconvey — BDD with a live-reload web UI. Use when you want browser-based reporting during development.
- matryer/is — a tiny assertion helper. Use when you want near-zero-dependency minimalism over any framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| Genesis | 2013-08 | Created at Pivotal by Onsi Fakhouri alongside Cloud Foundry[^1]. |
| 1.x | 2014–2021 | Original BDD DSL; long-lived stable line, still pinned by legacy suites. |
| 2.0.0 | 2022 | Major rewrite: two-phase tree, interruptible nodes, `Ordered`/`Serial`, decorators; custom reporters and `Measure` removed[^2]. |
| 2.x | 2022–present | Ongoing under import path `github.com/onsi/ginkgo/v2`; richer decorators and reporting. |

## References

[^1]: Ginkgo repository and documentation, Onsi Fakhouri. https://onsi.github.io/ginkgo/
[^2]: "Migrating to Ginkgo V2." https://onsi.github.io/ginkgo/MIGRATING_TO_V2
[^3]: Ginkgo docs — "Spec Parallelization." https://onsi.github.io/ginkgo/#spec-parallelization
[^4]: Kubernetes end-to-end tests are built on Ginkgo and Gomega. https://github.com/kubernetes/kubernetes/tree/master/test/e2e

## Tags

go, golang, testing, bdd, test-framework, gomega, integration-testing, end-to-end, test-parallelization, cli
