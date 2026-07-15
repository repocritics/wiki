# golangci/golangci-lint

> A runner that executes a hundred-plus Go linters in one pass, sharing a single type-checked program across them for speed.

[GitHub repo](https://github.com/golangci/golangci-lint) ·
[Official website](https://golangci-lint.run) ·
[License: GPL-3.0](https://github.com/golangci/golangci-lint/blob/main/LICENSE)

## Overview

golangci-lint is not itself a linter — it is an aggregator that bundles and runs a large set of third-party Go linters under one binary, one config file, and one output format. Its reason to exist is performance: loading and type-checking Go source is the expensive part of static analysis, so running `staticcheck`, `govet`, `errcheck`, and dozens of other analyzers as separate processes means paying that cost once per linter. golangci-lint loads and type-checks the program a single time and shares the result across every analyzer that speaks the `go/analysis` framework[^1]. On a large codebase this is the difference between a CI step that takes seconds and one that takes minutes.

It is effectively the default lint runner for the Go ecosystem: most Go projects that lint in CI do so through golangci-lint and its official GitHub Action rather than by wiring up analyzers by hand. The original author was Denis Isaev (jirfag); since roughly 2020 the project has been maintained primarily by Ludovic Fernandez (ldez)[^2].

The central tension is version drift. Because the tool aggregates linters it does not control, every release can bundle newer linter versions that surface new findings — a build that was green yesterday fails today after a routine upgrade. Pinning the exact version in CI is not optional hygiene here; it is load-bearing. The v2 release (2025) also broke config format compatibility, which every existing project had to migrate through[^3].

## Getting Started

```bash
# install a pinned version (recommended over `go install @latest`)
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/HEAD/install.sh \
  | sh -s -- -b $(go env GOPATH)/bin v2.1.0

golangci-lint run ./...
```

```yaml
# .golangci.yml  (v2 config format — note `version: "2"`)
version: "2"
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - unused
    - revive
  settings:
    revive:
      rules:
        - name: exported
issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck
```

Suppress a single finding inline with a `//nolint` directive:

```go
result, _ := doThing() //nolint:errcheck // handled by caller
```

## Architecture / How It Works

The core loop is: discover packages with `go/packages`, type-check them once, then run every enabled analyzer over the shared, already-loaded program. Linters fall into two groups. Those built on `golang.org/x/tools/go/analysis` (govet, staticcheck, ineffassign, and most others) are run together against the shared type information — this is where the speed comes from. Linters that need their own view of the source (formatters like gofmt/goimports, or tools with bespoke loaders) run in their own passes and cannot share the type-checked program.

Results are cached. golangci-lint hashes inputs (source files, config, linter set, build tags) and stores per-linter results, so an unchanged package is not re-analyzed on the next run. This cache is what makes local iteration and warm CI runs fast; a cold cache pays the full type-check cost.

Configuration is layered: a YAML/TOML/JSON file (`.golangci.yml` by convention) selects which linters are enabled, tunes each via a settings block, and defines exclusion rules. Only a small default set runs when nothing is configured — in v2 that is errcheck, govet, ineffassign, staticcheck, and unused. Enabling everything (`enable-all`) is discouraged because the full set is inconsistent and shifts between releases.

A frequent source of confusion is the `typecheck` pseudo-linter. When your code does not compile, golangci-lint cannot type-check it, and the compiler errors are reported under the name `typecheck`. It looks like a linter you should be able to disable, but it is really the build failing — the fix is to make the code compile, not to silence it.

## Production Notes

**Pin the version.** In CI, use `golangci/golangci-lint-action` with an explicit `version:` and never `latest`[^4]. A minor upgrade can enable new linters or bump a bundled linter (staticcheck especially), producing new findings that fail a previously passing build with no code change on your side.

**Memory, not CPU, is the scaling wall.** Because type information for the loaded packages is held in memory and shared across analyzers, peak RSS on a large monorepo can reach several GB. On memory-constrained CI runners this manifests as OOM kills rather than slowness. Lowering `--concurrency`, tuning `GOGC`, or linting subsets of packages are the usual mitigations.

**Bundled linter versions lag upstream.** golangci-lint ships a specific pinned version of each linter it embeds — you cannot independently upgrade only staticcheck to a newer release. If you need a linter's latest behavior before golangci-lint bundles it, you run that linter separately.

**Build tags and vendoring must match your build.** Analysis runs against the packages as configured; a mismatched `build-tags` setting silently drops files from analysis (missing findings) or pulls in the wrong ones (spurious findings). Getting this wrong is a quiet failure, not a loud one.

**The v1 to v2 migration is a config break.** v2 restructured the file: `linters-settings` moved under `linters.settings`, formatters (gofmt, goimports, gofumpt) split into a dedicated `formatters` section, and output config changed. The bundled `golangci-lint migrate` command auto-converts a v1 config, but the result should be reviewed rather than trusted blindly[^3].

**Deprecations churn.** Linters get deprecated as the ecosystem moves — `golint`, `deadcode`, `varcheck`, `structcheck`, `scopelint`, and others have been retired or folded into replacements over the years. A long-lived config will accumulate deprecation warnings and eventually errors.

## When to Use / When Not

**Use when:**
- You lint Go in CI and want one binary, one config, and one report instead of orchestrating analyzers by hand.
- You want the caching and shared-type-check speedup across many linters.
- You want consistent `//nolint` suppression and exclusion rules across the whole linter set.

**Avoid / reconsider when:**
- You only ever run one analyzer (e.g. just staticcheck) — the aggregator adds configuration surface and version-coupling you do not need.
- You need a linter's newest upstream version immediately and cannot wait for it to be bundled.
- You need vulnerability/CVE scanning of dependencies — that is govulncheck's job, not golangci-lint's.

## Alternatives

- dominikh/go-tools — the staticcheck suite; use directly when you want its high-signal checks without the aggregator and its version coupling.
- mgechev/revive — configurable style linter and the standard replacement for the deprecated golint; use when style rules are your main concern.
- securego/gosec — security-focused analyzer; use standalone when you want security scanning decoupled from your general lint run (it is also bundled).
- golang/tools (go vet) — ships with the toolchain; use when you want a zero-dependency baseline with no external installs.
- golang/vuln (govulncheck) — use for dependency CVE scanning, which golangci-lint deliberately does not cover.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2018-05 | First release; grew out of the GolangCI hosted service[^2]. |
| 1.0 | 2019-2020 | Long-lived v1 line; became the de facto Go lint runner. |
| 1.x | 2020-2024 | ldez lead maintainer; steady linter additions/deprecations[^2]. |
| 2.0 | 2025-03 | Breaking config format (`version: "2"`), formatters split out, `migrate` command added[^3]. |

## References

[^1]: golangci-lint README, "runs linters in parallel, uses caching … includes over a hundred linters." https://github.com/golangci/golangci-lint
[^2]: golangci-lint documentation and contributor history. https://golangci-lint.run
[^3]: golangci-lint v2 migration guide. https://golangci-lint.run/product/migration-guide/
[^4]: golangci/golangci-lint-action, version pinning guidance. https://github.com/golangci/golangci-lint-action

## Tags

go, golang, linter, static-analysis, ci, code-quality, developer-tools, lint-runner, cli, gpl
