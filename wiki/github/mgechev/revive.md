# mgechev/revive

> A configurable, extensible linter for Go, built as a drop-in replacement for the now-frozen golint.

[GitHub repo](https://github.com/mgechev/revive) ·
[Official website](https://revive.run) ·
[License: MIT](https://github.com/mgechev/revive/blob/master/LICENSE)

## Overview

Revive is a linter for Go written by Minko Gechev, first published in 2017[^1]. Its
original pitch was narrow: `golint`, the semi-official style linter maintained under
`golang/lint`, could not be configured — every rule was on, always, with no way to
tune, disable, or extend. Revive reimplements golint's rule set on top of a
configurable engine, so teams can pick exactly which checks run and with what
thresholds. When the Go team deprecated and froze `golint` in 2020[^2], revive became
the de facto successor and today ships inside most Go CI pipelines by way of
golangci-lint.

The defining design choice is that revive is a *rule framework*, not a fixed check
list. Each rule is a small Go type implementing a common interface that walks the
`go/ast` for a file and emits failures; the runner loads a set of rules from TOML
configuration and applies them. This makes revive easy to extend with project-specific
rules, but it also means revive is a *style and convention* linter rather than a bug
finder — it reasons mostly about naming, structure, and idiom, not about program
semantics. For deep correctness analysis (nil derefs, unused writes, concurrency bugs)
staticcheck remains the complementary tool.

The advertised "~6x faster than golint" figure is real but conditional: it applies when
type-checking rules are disabled[^1]. Most of revive's rules operate on the AST alone
and need no type information; a minority (for example `context-keys-type`) require the
`go/types` checker, and enabling those brings back the compile-and-typecheck cost that
made golint slow. Speed is therefore a function of which rules you turn on, not a fixed
property.

## Getting Started

```bash
# Homebrew
brew install revive

# or from source
go install github.com/mgechev/revive@latest
```

Minimal TOML config (`revive.toml`) enabling a few rules:

```toml
ignore-generated-header = false
severity = "warning"
confidence = 0.8

[rule.exported]
[rule.var-naming]
[rule.error-strings]
[rule.cyclomatic]
  arguments = [10]
```

```bash
revive -config revive.toml -formatter friendly ./...
```

By default — with no config — revive behaves like golint: it enables the golint rule set
and prints golint-style output, so it is a true drop-in. The moment you pass a config
file, only the rules you name are active (unless you set `enable-default-rules` or
`enable-all-rules`).

## Architecture / How It Works

A run is: parse packages → build a lint context → apply each configured rule to each
file → format the collected failures.

- **Rules** implement a `Rule` interface (`Name()` plus an `Apply(file, arguments)`
  that returns `[]Failure`). Most rules construct an `ast.Visitor` and walk the file.
  Because the unit of work is one file's AST, rules are cheap and embarrassingly
  parallel, which is where the speed comes from.
- **Typed vs untyped rules.** Rules declare whether they need type information. Untyped
  rules run against the syntax tree only. Typed rules pull in `go/types`, forcing a full
  type-check of the package's dependency graph — the expensive path. The README's rule
  table marks which rules are typed.
- **Configuration** is TOML. Top-level keys set defaults (`severity`, `confidence`,
  exit codes); `[rule.<name>]` blocks enable and tune individual rules with `arguments`,
  `severity`, `disabled`, and per-rule `exclude` globs. `confidence` is a per-failure
  threshold: rules emit failures with a confidence score and anything below the cutoff
  is dropped.
- **Comment directives** let source files opt out inline: `//revive:disable`,
  `//revive:enable`, `//revive:disable-line`, `//revive:disable-next-line`, optionally
  scoped to a rule (`//revive:disable:unexported-return`). Config can *force* those
  directives to carry a reason or a rule name via `[directive.specify-disable-reason]`
  and `[directive.specify-disable-rule]`.
- **Formatters** are pluggable and cover human output (`friendly`, `stylish`,
  `default`) and machine output (`json`, `ndjson`, `checkstyle`, `unix`, and SARIF for
  code-scanning integrations). Note that `stylish` buffers all output before printing, so
  it can feel slower than streaming formatters on large runs.

Revive is also consumable as a Go library (`github.com/mgechev/revive/lint`), which is
how aggregators embed it and how you register custom rules or formatters without forking.

## Production Notes

- **Most people run revive *through* golangci-lint, not standalone**, and the two use
  different config formats — revive's own config is TOML, but when embedded it is
  configured under a `linters.settings.revive` block in golangci-lint's YAML. A
  `revive.toml` sitting in the repo is silently ignored in that setup. This mismatch is
  the single most common source of "my revive config isn't being applied" confusion.
- **Enablement semantics are a footgun.** With a config present, revive enables *only*
  the rules you list — it does not merge with golint defaults. Teams migrating from
  golint often see far fewer warnings than expected until they add
  `enable-default-rules = true`. `enable-all-rules` and `enable-default-rules` cannot be
  combined.
- **Style rules are noisy on legacy code.** `exported`, `var-naming`, and
  `package-comments` will flood an established codebase that predates these conventions.
  The practical path is to start from the README's recommended rule set, adopt
  incrementally, and lean on `confidence` and per-rule `exclude` globs (e.g. `**/*.pb.go`,
  the built-in `TEST` shortcut) rather than blanket-disabling.
- **Typed rules gate your speed.** If a fast lint matters in CI, keep type-requiring
  rules out of the hot path; enabling `enable-all-rules` pulls them in and erases the
  performance advantage over golint.
- **Exit-code control is explicit.** `error-code`/`warning-code` default to `0`, so a
  bare `revive` reports findings but does not fail the build. Use `-set_exit_status` (or
  set the codes) to make CI actually break on findings — easy to forget.

## When to Use / When Not

**Use when:**
- You want configurable, golint-compatible style/idiom checks for Go.
- You need project-specific lint rules and want to write them without forking a tool.
- You're standardizing conventions (naming, comments, complexity limits) across a team.
- You already run golangci-lint and want its style-linting slot filled by something tunable.

**Avoid when:**
- You need semantic bug detection (dead stores, nil analysis, concurrency issues) —
  that's staticcheck / go vet territory, not revive's.
- You want zero configuration and zero opinions: `gofmt`/`gofumpt` cover formatting with
  no rule tuning at all.
- You only care about security findings — reach for a dedicated SAST tool.

## Alternatives

- golangci/golangci-lint — the aggregator that bundles revive alongside dozens of
  linters; use it when you want one runner and cache for the whole suite (and revive is
  usually a component of it, not a competitor).
- dominikh/go-tools — staticcheck; use it when you need real bug/correctness analysis
  rather than style and convention checks.
- golang/lint — the original golint; do not use it for new work, it is frozen and
  deprecated in favor of tools like revive.
- securego/gosec — use it when the goal is security scanning (SQL injection, weak crypto,
  unhandled errors with security impact) rather than style.
- mvdan/gofumpt — use it when you want stricter deterministic formatting enforced rather
  than configurable lint rules.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-07 | First public release: configurable golint reimplementation on a rule framework[^1]. |
| — | 2020 | Go team freezes/deprecates `golint`; revive becomes the de facto successor[^2]. |
| v1.x | current | Stable v1 line: ~90 rules, TOML config, comment directives, and formatters including SARIF. Actively maintained (last push 2026-07)[^3]. |

## References

[^1]: revive README — feature list, golint comparison, and speed claim. https://github.com/mgechev/revive
[^2]: `golang/lint` — deprecation notice ("Golint is deprecated... frozen"). https://github.com/golang/lint
[^3]: revive releases. https://github.com/mgechev/revive/releases

## Tags

go, golang, linter, static-analysis, code-quality, golint, developer-tools, cli, ast, style-checker
