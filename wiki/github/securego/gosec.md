# securego/gosec

> Static analyzer that inspects Go source for security problems by walking the AST and SSA representation.

[GitHub repo](https://github.com/securego/gosec) ·
[Official website](https://securego.io) ·
[License: Apache-2.0](https://github.com/securego/gosec/blob/master/LICENSE.txt)

## Overview

gosec is a security-focused static analysis tool for Go. It scans source code for a fixed catalog of known-bad patterns — hardcoded credentials, weak crypto, SQL string concatenation, unhandled errors on security-relevant calls, insecure file permissions, TLS misconfiguration — and reports each finding with a severity, a confidence level, and a CWE mapping[^1]. It began life as `gas` (Go AST Scanner) and was renamed to gosec around 2018; the current module is versioned as `github.com/securego/gosec/v2`[^2].

The tool sits in the "SAST for one language" niche. Its rules are hand-written Go visitors keyed to specific standard-library and language constructs, which makes it fast, dependency-light, and precise on the patterns it knows — but also inherently incomplete. gosec finds what its rule authors anticipated; it is not a general dataflow engine of the depth of CodeQL. The defining tension is between that curated, low-friction rule set and the false positives that pattern matching produces: a large fraction of real-world usage is spent tuning severity/confidence thresholds and annotating verified-safe lines with `#nosec`.

More recent releases have pushed beyond pure pattern matching. gosec now ships SSA-based analyzers for type conversions, slice bounds, and some crypto checks, plus a taint-analysis rule family (the `G7xx` group) that tracks data flow from user input to dangerous sinks — SQL injection, command injection, path traversal, SSRF, XSS, and others[^1]. This narrows the gap with heavier tools but does not close it. The project is actively maintained, with commits landing regularly as of mid-2026.

## Getting Started

gosec requires Go 1.25 or newer.

```bash
go install github.com/securego/gosec/v2/cmd/gosec@latest
```

```bash
# Scan every package in the current module
gosec ./...

# Emit SARIF for GitHub code scanning, don't fail the build
gosec -no-fail -fmt sarif -out results.sarif ./...

# Only fail on medium-or-higher severity findings
gosec -severity medium ./...
```

As a GitHub Action:

```yaml
- name: Run Gosec Security Scanner
  uses: securego/gosec@master   # tag tracks the latest stable release
  with:
    args: -no-fail -fmt sarif -out results.sarif ./...
```

Exit code is `0` when no unsuppressed findings remain, `1` otherwise; `-no-fail` forces `0` (the usual choice when a downstream SARIF upload owns the pass/fail decision).

## Architecture / How It Works

gosec loads packages through Go modules (via `golang.org/x/tools/go/packages`), so it type-checks the code it scans and needs dependencies resolvable — a scan on a project with a broken build or missing modules will error or degrade. Rules operate over two representations:

- **AST rules** — most of the catalog. Each rule is a visitor matched against specific node shapes: a call to `exec.Command`, an `http.Transport` with `InsecureSkipVerify: true`, an `os.OpenFile` with permissive mode bits. Fast, but syntactic — they see the shape, not the data.
- **SSA / taint rules** — the `G7xx` family and several analyzers build on the SSA form to follow values across assignments and function boundaries, flagging a sink only when a source can reach it. This reduces false positives on injection classes at the cost of scan time.

Rules are grouped by ID prefix: `G1xx` general (credentials, unsafe, HTTP/cookie hardening), `G2xx` injection in query/template/command construction, `G3xx` file and path handling, `G4xx` crypto and TLS, `G5xx` blocklisted imports, `G6xx` Go-specific correctness/security, `G7xx` taint analysis[^1]. Every finding carries a CWE identifier via a static mapping in the source[^3].

Suppression works at two levels. Externally, `-include=`/`-exclude=` select rule subsets and `exclude-rules` scopes exclusions to path regexes. Inline, a `#nosec [RuleList] [-- justification]` comment (or the equivalent `//gosec:disable`) on the reporting line silences a finding. A `goanalysis` package also exposes gosec as a standard `analysis.Analyzer`, letting it plug into tools like Bazel's nogo.

## Production Notes

**Most teams run gosec through golangci-lint, not standalone.** golangci-lint bundles gosec as one linter among many and often pins an older gosec version than upstream; rule behavior and defaults can therefore differ from what the standalone binary produces. If a finding appears or disappears unexpectedly, check which gosec version your aggregator ships before assuming a code change caused it.

**False positives are the operational reality.** `#nosec` is used heavily in practice, and unscoped naked directives are a known footgun — one `#nosec` on a line silences every rule there, not just the intended one. The opt-in `-nosec-require-rules` and `-nosec-require-justification` flags reject naked or unexplained directives; turning both on is the recommended discipline for a shared codebase, but they default off so existing repos are unaffected.

**Scan cost tracks package loading and SSA.** Because gosec type-checks through Go modules, cold scans of large repos are dominated by package loading, not rule evaluation. Some rules resolve the project's Go version with `go list`, which can be slow; setting `GOSECGOVERSION=go1.21.1` (or similar) skips that lookup. The `G7xx` taint rules add real time on large codebases — scope them out where you don't need them.

**Test files and vendored code are skipped by default.** Pass `-tests` to include `_test.go` files; vendor directories are always ignored. `-exclude-generated` drops files carrying the standard `// Code generated ... DO NOT EDIT.` marker. Forgetting these defaults is a common "why didn't it flag my test helper" surprise.

**SARIF/JSON only for suppression tracking.** `-track-suppressions` records why each finding was silenced, but only the SARIF and JSON formatters emit that metadata; text/CSV/HTML output drops it. If an audit trail of suppressions matters, pin the output format accordingly.

**The AI auto-fix feature calls out to a third-party API.** Newer gosec can request fix suggestions from an external LLM provider (Atlas Cloud, Gemini, Claude, OpenAI, or any OpenAI-compatible endpoint). It is opt-in and off by default, but be aware that enabling it sends code context off-machine — a non-starter in many regulated or air-gapped environments.

## When to Use / When Not

**Use when:**
- You want a fast, zero-config first-pass security linter for a Go codebase, ideally wired into CI via SARIF + GitHub code scanning.
- You already run golangci-lint and want gosec's rules included cheaply.
- You need CWE-tagged findings for compliance reporting without standing up a heavier platform.

**Avoid (or supplement) when:**
- You need deep interprocedural dataflow across your whole program — CodeQL or Semgrep's dataflow go further than gosec's taint rules.
- Your concern is vulnerable dependencies or supply chain rather than your own source patterns — that is a different tool class (see below).
- You cannot tolerate the false-positive tuning overhead, or you need custom rules in a first-class DSL (gosec rules are Go code, not a declarative pattern language).

## Alternatives

- golangci-lint/golangci-lint — meta-linter that bundles gosec plus dozens of others; use it when you want gosec's rules alongside general Go linting in a single run.
- github/codeql — semantic, cross-function taint analysis; use when you need dataflow depth beyond gosec's rule catalog and already live in GitHub code scanning.
- semgrep/semgrep — multi-language, declarative pattern/dataflow rules; use when you want custom, readable rules or coverage beyond Go.
- aquasecurity/trivy — dependency, container, and IaC vulnerability scanning; use when the risk is known-vulnerable components rather than insecure source patterns.
- google/osv-scanner — flags dependencies with known OSV/CVE advisories; complementary to gosec, not a replacement for source-level rules.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-07 | Repository created as `gas` (Go AST Scanner)[^2]. |
| rename | ~2018 | Project renamed from `gas` to `gosec`[^2]. |
| v2.x | 2019 | Module path moved to `github.com/securego/gosec/v2`; Go modules support. |
| ongoing | 2020–2024 | SARIF output, CWE mapping, expanded rule catalog, SSA-based analyzers. |
| recent | 2025–2026 | Taint-analysis rules (`G7xx`), opt-in `#nosec` strictness flags, AI auto-fix, Go 1.25 baseline. |

## References

[^1]: gosec README — features, rule categories (G1xx–G7xx), and taint analysis. https://github.com/securego/gosec
[^2]: gosec module path `github.com/securego/gosec/v2` and project origins as `gas`. https://pkg.go.dev/github.com/securego/gosec/v2
[^3]: CWE mapping defined in the source. https://github.com/securego/gosec/blob/master/issue/issue.go

## Tags

go, golang, security, static-analysis, sast, taint-analysis, code-scanning, sarif, cli, devsecops
