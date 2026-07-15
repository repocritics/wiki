# vektra/mockery

> A code generator that produces testify-compatible mock implementations for Go interfaces, removing hand-written mock boilerplate.

[GitHub repo](https://github.com/vektra/mockery) ·
[Official website](https://vektra.github.io/mockery/) ·
[License: BSD-3-Clause](https://github.com/vektra/mockery/blob/v3/LICENSE)

## Overview

mockery reads Go interface declarations and emits mock structs that satisfy
those interfaces, wired to the `stretchr/testify/mock` runtime. Instead of
hand-writing a struct that records calls and returns canned values for each
method, you point mockery at a package and it generates one mock type per
interface. The generated mocks integrate with testify's assertion model
(`On(...)`, `Return(...)`, `AssertExpectations(t)`), so teams already using
testify get mocks for free.

The project is old by Go standards — the repository dates to 2014[^1] — and
has been through two large redesigns. v2 moved configuration into a
`.mockery.yaml` file and introduced the type-safe "expecter" API
(`m.EXPECT().Method(...)`), replacing the earlier reliance on stringly-typed
`On("Method", ...)` calls. v3 is a ground-up rewrite around user-editable Go
templates[^2]: the mock body is produced by a template, and mockery ships
several (the default testify template plus alternatives), which lets a project
change the shape of its generated mocks without patching the tool.

The defining tension is churn versus stability. mockery solves a real problem
well, but its configuration surface has been rewritten twice, and each rewrite
broke the previous invocation style. A codebase that adopted mockery in the v1
or early-v2 era will not upgrade to v3 without editing config and regenerating
every mock. That cost is the main thing to weigh before adopting it.

## Getting Started

```bash
# Install the v3 line (module path is versioned)
go install github.com/vektra/mockery/v3@v3
```

Configure with a `.mockery.yml` at the repo root and list the packages to mock:

```yaml
# .mockery.yml (v3)
template: testify
packages:
  github.com/you/project/store:
    config:
      all: true
```

Then generate and use the mock in a test:

```go
func TestHandler(t *testing.T) {
    m := NewMockStore(t)                 // t wired in; auto-asserts on cleanup
    m.EXPECT().Get("key").Return("v", nil).Once()

    got, err := m.Get("key")
    require.NoError(t, err)
    require.Equal(t, "v", got)
}
```

Run `mockery` (no args) to regenerate all configured mocks.

## Architecture / How It Works

mockery is a static analyzer plus a template engine. It loads packages through
Go's `go/packages` toolchain (the same machinery `go vet` uses), walks the
type-checked AST to find interface declarations, and builds an in-memory model
of each interface's method set — names, parameters, variadics, return types,
and the imports those types require. That model is handed to a template that
renders the mock source, and the output is run through `go/format` (or
goimports-style import fixing) so it compiles cleanly.

Because it type-checks rather than text-parses, mockery resolves embedded
interfaces, aliased types, and generics correctly, at the cost of needing the
target package to actually build. If your code does not compile, mockery cannot
generate mocks for it.

The generated testify mocks are thin: each method calls `m.Called(args...)` on
an embedded `mock.Mock`, then unpacks the configured return values. The
expecter layer (`EXPECT()`) is a generated, strongly-typed wrapper over
testify's stringly-typed `On`/`Return`, giving compile-time checking of method
names and argument arity that raw testify lacks.

v3's template-first design is the structural change worth understanding: the
"what a mock looks like" decision moved out of Go code and into templates a
project can select or replace. This decouples mockery's parser from any single
mock style, but it also means the v3 config schema (`template`, `packages`,
per-package `config`) does not map one-to-one onto v2's, which is why the
upgrade is not a version bump.

## Production Notes

**Generated code should be committed and CI-verified.** The standard failure
mode is drift: someone changes an interface, forgets to regenerate, and mocks
silently compile against the old shape (or stop compiling). The mitigation is a
CI step that runs mockery and fails if `git diff` is non-empty — treat mocks
like any other generated artifact.

**The v2 → v3 migration is the real cost.** v3 removed most legacy CLI flags
(`--name`, `--all`, `--dir`, `--inpackage` and friends) and the older config
keys in favor of the `packages` map. Large monorepos have hundreds of mock
files and dozens of per-package settings; budget real time, pin the mockery
version in your toolchain, and migrate package-by-package. Do not assume a
v2 `.mockery.yaml` works under v3.

**Version pinning matters.** Because generated output can shift between
mockery releases (formatting, template tweaks), an unpinned `@latest` in CI
can produce spurious diffs. Pin the exact version in a tools file
(`tools.go` / `go.mod` tool directive) or your Taskfile so every developer and
CI runner generates byte-identical output.

**testify coupling is a feature and a constraint.** The default mocks depend on
`stretchr/testify/mock`. That is fine if your suite already uses testify, but
it pulls testify into the dependency graph of any package that imports the
mocks. Teams wanting dependency-free fakes (plain func fields, no assertion
runtime) are a poor fit for the default template and are usually happier with a
moq-style generator.

**Interfaces must be mockable interfaces.** mockery mocks interface types, not
concrete structs. Code that depends directly on concrete types has to be
refactored to accept interfaces first; mockery does not paper over that.

## When to Use / When Not

**Use when:**
- Your test suite already uses `stretchr/testify` and you want matching mocks.
- You mock many interfaces and hand-writing them is real toil.
- You want compile-time-checked expectations via the `EXPECT()` API.
- You can commit generated code and enforce regeneration in CI.

**Avoid when:**
- You want zero test-time dependencies — moq's func-field fakes are leaner.
- You prefer the gomock controller/`EXPECT` idiom and interaction ordering.
- Your project can't tolerate config churn — mockery has rewritten its
  configuration twice.
- You only have a handful of small interfaces; hand-written stubs may be simpler
  than adopting a generator and its config file.

## Alternatives

- uber-go/mock — the maintained successor to golang/mock (gomock); use it when you prefer a controller-based API with call-ordering assertions over testify.
- matryer/moq — generates simple struct-with-func-field fakes and no runtime dependency; use when you want dependency-free mocks and less machinery.
- golang/mock — the original gomock, now archived/deprecated; only relevant to legacy code that hasn't moved to uber-go/mock.
- maxbrunsfeld/counterfeiter — generates "fake" implementations in a counterfeiter idiom; use when your team already standardizes on that style.
- stretchr/testify — provides the `mock.Mock` base that mockery generates against; use it directly when you'd rather hand-write a few mocks than run a generator.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014 | Repository created; early command-line mock generator[^1]. |
| v2 | ~2020 | YAML `.mockery.yaml` config, `packages` feature, type-safe `EXPECT()` expecter API. |
| v3 | 2025 | Template-based rewrite; legacy CLI flags removed, config schema reworked around templates + packages[^2]. |

## References

[^1]: vektra/mockery repository, created 2014-09-02. https://github.com/vektra/mockery
[^2]: mockery documentation (v3), templates and configuration. https://vektra.github.io/mockery/
[^3]: stretchr/testify mock package. https://pkg.go.dev/github.com/stretchr/testify/mock

## Tags

go, golang, testing, mocks, code-generation, testify, mock-generator, developer-tools, unit-testing, static-analysis
