# gitleaks/gitleaks

> A regex-and-entropy secret scanner for git history, files, and stdin — now feature-frozen and in security-patch-only mode.

[GitHub repo](https://github.com/gitleaks/gitleaks) ·
[Official website](https://gitleaks.io) ·
[License: MIT](https://github.com/gitleaks/gitleaks/blob/master/LICENSE)

## Overview

Gitleaks is a Go CLI that finds hardcoded secrets — API keys, tokens, passwords, private keys — in git repositories, directories, and piped input. It was created by Zachary Rice (`zricethezav`) in 2018 and became one of the default answers to "how do I stop committing credentials," largely because it ships a single static binary, a batteries-included ruleset, and a `.gitleaks.toml` format that teams can extend. As of 2026 it has ~28k stars and is embedded in a large fraction of CI pipelines and pre-commit setups[^1].

The detection engine is deliberately unglamorous: per-rule regular expressions, a keyword pre-filter to skip content that cannot match, and optional Shannon-entropy thresholds to suppress low-randomness false positives. The maintainer's own framing is "regex is (almost) all you need"[^2]. This is the defining tension of the tool: it is fast and dependency-light, but it detects *patterns*, not *valid* secrets — it does not call provider APIs to confirm a key is live, so both false positives and expired-but-flagged secrets are routine.

The most important fact for anyone adopting it in 2026 is the project status. The README now carries a maintainer warning that Gitleaks is **feature complete**: no new features will be merged, future releases are security patches only, and the author has shifted focus to a successor project, Betterleaks[^3]. Gitleaks still works and is still maintained defensively, but new rules and capabilities are no longer arriving here.

## Getting Started

```bash
# macOS
brew install gitleaks

# Docker
docker pull ghcr.io/gitleaks/gitleaks:latest

# From source (Go toolchain required)
git clone https://github.com/gitleaks/gitleaks.git && cd gitleaks && make build
```

Scan a repo's full history, printing findings verbosely:

```bash
gitleaks git -v /path/to/repo
```

Scan a working directory (not git-aware) or pipe arbitrary content:

```bash
gitleaks dir -v /path/to/files
cat some_file | gitleaks stdin -v
```

Wire it into pre-commit so secrets are blocked before they land:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.2
    hooks:
      - id: gitleaks
```

## Architecture / How It Works

Gitleaks has three scan modes, unified in v8 under explicit subcommands: `git` (scans patches via `git log -p`, controllable with `--log-opts`), `dir` (walks files/directories, no history), and `stdin`. A finding carries the secret, the matched `RuleID`, computed entropy, file/line/commit provenance, and a stable `Fingerprint` used for deduplication and ignore-lists.

The rule pipeline for each content fragment is: (1) keyword pre-check — if a rule declares `keywords`, the fragment is only regex-tested when a keyword string is present, which is the main performance lever on large repos; (2) regex match using Go's RE2 engine — note RE2 has **no lookahead/lookbehind**, which constrains how rules can be written; (3) optional entropy gate on a capture group; (4) allowlist evaluation. Configuration is TOML: you either author a standalone config or `[extend] useDefault = true` to inherit the ~hundreds of built-in provider rules and layer your own `[[rules]]` on top. Config resolution order is `--config` → `GITLEAKS_CONFIG` → `GITLEAKS_CONFIG_TOML` → `.gitleaks.toml` in the target → built-in default.

False-positive management is the part most teams actually live in. There are inline `gitleaks:allow` comments, a `.gitleaksignore` file keyed by fingerprint, and `[[allowlists]]` blocks scoped globally, per-rule, or to a `targetRules` set — each supporting `commits`, `paths`, `regexes` (with `regexTarget` of secret/match/line), and `stopwords`, combined with `AND`/`OR` conditions. For large histories, a **baseline** (`--baseline-path`) diffs against a prior report so only new findings surface. v8.28.0 added **composite rules** (`[[rules.required]]` with `withinLines`/`withinColumns` proximity), letting a primary match require nearby auxiliary matches to fire — a structural answer to noisy single-regex rules[^4].

## Production Notes

- **It matches patterns, not live secrets.** Gitleaks will happily flag a rotated, revoked, or example key, and it misses secrets that don't fit a known shape. If your workflow depends on "is this credential actually valid," Gitleaks alone won't tell you — this is the single biggest source of alert fatigue.
- **Scanning deep history is expensive.** `gitleaks git` re-reads every patch. On large monorepos this is minutes, not seconds. Baselines and `--log-opts="commitA..commitB"` ranges are the standard mitigations; in CI, scan only the pushed range rather than `--all` every run.
- **`generic-api-key` is the noisy one.** The catch-all high-entropy rule generates most false positives. Many teams disable it (`disabledRules = ["generic-api-key"]`) and rely on the specific provider rules, accepting reduced recall for a usable signal.
- **Command deprecation bites CI.** v8.19.0 hid the old `detect`/`protect` commands in favor of `git`/`dir`/`stdin`. Pinned pipelines calling `gitleaks detect` still work but are undocumented; migrate to the new verbs[^5].
- **Config schema has shifted across minors.** `[rules.allowlist]` → `[[rules.allowlists]]` (v8.21.0) and `[allowlist]` → `[[allowlists]]` (v8.25.0) were backwards-compatible but easy to trip over when copying old configs. Verify your config against the version you pin.
- **Pin the version.** Because the default ruleset changes between releases, an unpinned `latest` in CI or pre-commit can silently change which findings appear. Pin `rev:`/image tags and upgrade deliberately.
- **`gitleaks-action` licensing.** The GitHub Action requires a license key for organizations (free for individuals/OSS); the CLI itself is MIT and unencumbered[^6].
- **Feature-frozen.** New provider rules will not land upstream anymore. Plan to maintain your own custom rules, or evaluate the successor/alternatives below, if you need coverage for newer credential formats.

## When to Use / When Not

**Use when:**
- You want a fast, self-contained secret scan in pre-commit or CI with zero runtime dependencies.
- You need to audit historical commits, not just the current tree.
- You want to author custom regex rules and manage allowlists in version-controlled TOML.
- SARIF/JSON/CSV/JUnit report output needs to feed an existing security dashboard.

**Avoid (or supplement) when:**
- You need verified/live secret detection — pair with or switch to a tool that validates against provider APIs.
- You require ongoing new-feature development and fresh provider coverage; the project is frozen.
- You want managed policy, dashboards, and remediation workflows out of the box (that's a SaaS product, not this CLI).
- Your rules need lookahead/lookbehind — Go's RE2 doesn't support them.

## Alternatives

- trufflesecurity/trufflehog — verifies detected secrets by calling live provider APIs; use when false-positive volume from pattern-only matching is unacceptable.
- gitguardian/ggshield — CLI backed by GitGuardian's SaaS ruleset and dashboards; use when you want managed policy, historical monitoring, and centralized remediation.
- semgrep/semgrep — general-purpose SAST with secret rules; use when secret scanning is one part of broader static analysis you already run.
- awslabs/git-secrets — minimal git-hook-only scanner focused on AWS keys; use when you want the smallest possible pre-commit guard and don't need history scanning.
- betterleaks/betterleaks — the original author's successor project; consider when you want continued feature development in the same lineage.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2018-01 | Created by Zachary Rice (`zricethezav`); Go-based git history secret scanner[^1]. |
| v8.0 | 2022 | Major rewrite; unified `git`/`dir`/`stdin` model and current TOML config schema. |
| v8.6.0 | 2022 | `keywords` pre-filter for faster rule matching. |
| v8.19.0 | 2024 | `detect`/`protect` deprecated in favor of `git`/`dir`; hidden from help[^5]. |
| v8.21.0 | 2024 | `[[rules.allowlists]]` replaces `[rules.allowlist]` (backwards-compatible). |
| v8.25.0 | 2025 | Global `[[allowlists]]` + `targetRules`; `[allowlist]` deprecated. |
| v8.28.0 | 2025 | Composite/`required` rules with `withinLines`/`withinColumns` proximity[^4]. |
| — | 2026 | Declared feature complete; security patches only, author moves to Betterleaks[^3]. |

## References

[^1]: gitleaks/gitleaks repository (repo metadata, ~28.1k stars, MIT, Go, created 2018-01-27). https://github.com/gitleaks/gitleaks
[^2]: "Regex is (almost) all you need" — detection-engine writeup by the maintainer. https://lookingatcomputer.substack.com/p/regex-is-almost-all-you-need
[^3]: Maintainer notice in the project README: Gitleaks is feature complete; future releases are security patches only; focus shifting to Betterleaks. https://github.com/gitleaks/gitleaks#gitleaks
[^4]: Composite rules and proximity matching (`[[rules.required]]`, `withinLines`/`withinColumns`), introduced v8.28.0 — documented in the README configuration section. https://github.com/gitleaks/gitleaks#configuration
[^5]: Command deprecation note (v8.19.0 hides `detect`/`protect`) with migration gist. https://gist.github.com/zricethezav/b325bb93ebf41b9c0b0507acf12810d2
[^6]: gitleaks-action — GitHub Action wrapper with organizational licensing. https://github.com/gitleaks/gitleaks-action

## Tags

go, security, secret-scanning, devsecops, cli, static-analysis, git, ci-cd, pre-commit, credential-detection, regex
