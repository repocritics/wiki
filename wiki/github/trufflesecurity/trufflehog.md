# trufflesecurity/trufflehog

> A secrets scanner that doesn't just find candidate credentials — it logs into the service to confirm which ones are live.

[GitHub repo](https://github.com/trufflesecurity/trufflehog) ·
[Official website](https://trufflesecurity.com) ·
[License: AGPL-3.0](https://github.com/trufflesecurity/trufflehog/blob/main/LICENSE)

## Overview

TruffleHog finds leaked credentials — API keys, database passwords, private keys, tokens — across git history, filesystems, container images, cloud object stores, and SaaS platforms. It began as a Python tool by Dylan Ayrey that flagged high-entropy strings and regex matches in git history[^1]. The current codebase (v3) is a ground-up rewrite in Go, maintained by the company Truffle Security, and is one of the two dominant open-source secret scanners alongside Gitleaks.

Its defining feature is **active verification**. Most scanners stop at "this looks like an AWS key." TruffleHog goes further: for each of its 800+ detector types it makes a live API call to the credential's presumed owner (e.g. `GetCallerIdentity` for AWS) and reports whether the secret actually authenticates[^2]. This collapses the false-positive problem that makes entropy/regex scanners noisy — a "verified" result is an active, exploitable credential, not a guess. The tradeoff is that verification transmits found secrets to third-party APIs and depends on network reachability and rate limits, which is a real consideration in locked-down or air-gapped environments.

The second thing to know is the license: TruffleHog is **AGPL-3.0**, not the permissive MIT/Apache common to CLI tooling. For running the binary in CI this is inert, but it constrains anyone who wants to embed the engine into a hosted service. The company funds the OSS work with a separate closed Enterprise product that continuously monitors Git, Slack, Jira, Confluence, and similar sources[^3].

## Getting Started

```bash
brew install trufflehog
# or: docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest
# or: curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /usr/local/bin
```

Scan a git repo and print only credentials confirmed live:

```bash
trufflehog git https://github.com/trufflesecurity/test_keys --results=verified
```

```
Found verified result 🐷🔑
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIAYVP4CIPPERUVIFXG
Line: 4
Commit: fbc14303ffbf8fb1c2c1914e8dda7d0121633aca
File: keys
Repository: https://github.com/trufflesecurity/test_keys
```

Scan the whole working tree in CI and fail the build if anything verifies:

```bash
trufflehog git file://. --since-commit main --branch feature-1 --results=verified,unknown --fail
```

The `--fail` flag exits with code 183 when results are found, which is the hook most CI pipelines gate on.

## Architecture / How It Works

The engine is a fan-out of **sources → decoders → detectors → verifiers**:

1. **Sources** produce chunks of bytes. Each scan target is a subcommand: `git`, `github`, `gitlab`, `filesystem`, `docker`, `s3`, `gcs`, `postman`, `jenkins`, `elasticsearch`, `huggingface`, `syslog`, `stdin`, and more. The git source walks commit history so secrets deleted in later commits are still found.
2. **Decoders** normalize each chunk — plain text, base64, UTF-16, etc. — so a detector sees candidate strings regardless of encoding.
3. **Detectors** are the 800+ credential-type matchers under `pkg/detectors`. Each combines keyword prefiltering (cheap) with a regex to extract candidates. A detector is defined by its `Keywords()`, `FromData()` extraction, and a verifier.
4. **Verification** is the per-detector live check. AWS keys call STS; GitHub tokens call the API; a private key is checked against known public keys and TLS certificates via the project's Driftwood dataset[^4].

Results carry one of three statuses that matter for every downstream decision: **verified** (API confirmed live), **unverified** (matched but not confirmed, e.g. verification disabled or the credential is dead), and **unknown** (verification was attempted but errored — network/API failure). `--results=` selects which to emit; the default is `verified,unverified,unknown`.

Two subcommands are worth calling out. `github-experimental --object-discovery` enumerates deleted and hidden commits (Cross Fork Object References) that don't appear in normal history, then scans them — this is how TruffleHog surfaces secrets that authors thought they had erased[^5]. And `--no-verification` degrades the tool into a regex-only scanner for offline use.

## Production Notes

**Verification is a network side effect.** Every verified detector makes outbound calls to the credential's home service, using the credential itself. In a scan of a large monorepo this is thousands of authentication attempts against AWS, GitHub, Stripe, etc. Consequences: it can trip provider-side anomaly detection, it fails in air-gapped CI, and it leaks the fact-of-existence of secrets to third parties. Use `--no-verification` where that matters, and expect `unknown` results whenever the network is flaky.

**Rate limits dominate large scans.** Unauthenticated GitHub org scans are throttled hard; pass `--token` for a higher ceiling. The scan time is usually bound by source enumeration and verification round-trips, not detection — `github-experimental` object discovery in particular can take 20 minutes to several hours to enumerate a large repo's hidden commits before scanning even begins.

**Tune the noise, not just the results filter.** For unverified output the signal-to-noise depends on `--filter-entropy` (start around 3.0) and `--filter-unverified` (one result per chunk per detector). Teams that scan for `unverified` without these flags drown in candidates.

**CI ergonomics.** Exit code 183 on `--fail`, `--since-commit` / `--branch` for diffing PR ranges, and `--github-actions` / `--json` output formats. There is an official GitHub Action and a pre-commit hook. Inline `trufflehog:ignore` comments suppress a known false positive on a specific line for sources that expose line numbers.

**Local git scanning is sandboxed.** Since CVE-2025-41390, TruffleHog clones a local repo to a temp directory before scanning to defeat malicious `git config` in an untrusted checkout. `--trust-local-git-config` skips this (trusted repos only); `--clone-path` overrides the temp location[^6].

**License.** AGPL-3.0. Fine to run as a tool in your pipeline. If you plan to wrap the engine in a product or hosted service, get the license reviewed — this is the single most common reason organizations choose Gitleaks (MIT) instead.

## When to Use / When Not

**Use when:**
- You want low-false-positive results — "verified" means an attacker could use it right now.
- You need broad coverage across git, cloud buckets, containers, and SaaS in one tool.
- You're auditing history for leaked secrets, including deleted/hidden commits.
- You want a CI gate that fails only on credentials confirmed to be live.

**Avoid when:**
- You need a permissively licensed engine to embed in your own product (AGPL friction).
- You're scanning in an air-gapped or egress-restricted environment where verification can't reach provider APIs (you lose the main advantage).
- You want the fastest possible pre-commit hook on regex alone — Gitleaks is leaner for that narrow job.
- Provider-side security teams would treat thousands of auth attempts as an incident.

## Alternatives

- gitleaks/gitleaks — MIT-licensed regex+entropy scanner; faster and more embeddable, but no live verification, so more false positives. The default choice when license or offline operation matters.
- Yelp/detect-secrets — pre-commit-focused with a baseline file to track and accept known secrets over time; use when you want a review workflow rather than verification.
- awslabs/git-secrets — tiny, AWS-oriented pre-commit guard; use when you only need to block AWS keys from ever being committed.
- semgrep/semgrep — broader SAST that also finds secrets among many other rules; use when secret scanning is one part of a wider static-analysis program.
- GitHub secret scanning (native) — zero-setup and push-protection on GitHub-hosted repos; use when you're all-in on GitHub and don't need to scan non-git sources.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1/v2 (Python) | 2016–2021 | Original entropy + regex scanner of git history by Dylan Ayrey[^1]. |
| v3.0 | 2022 | Complete Go rewrite by Truffle Security; detectors with active verification, multi-source scanning[^2]. |
| — | 2022+ | Driftwood private-key verification, GitHub Action, pre-commit hook[^4]. |
| — | 2023–2024 | `github-experimental` object discovery for deleted/hidden commits[^5]. |
| — | 2025 | Local-git-config sandboxing (CVE-2025-41390)[^6]. |

## References

[^1]: Dylan Ayrey, original TruffleHog (Python) — high-entropy string and regex detection over git history. https://github.com/trufflesecurity/trufflehog
[^2]: TruffleHog README, "What's new in v3?" — Go rewrite, 700+ verifying detectors, multi-source scanning. https://github.com/trufflesecurity/trufflehog#whats-new-in-v3
[^3]: Truffle Security, TruffleHog Enterprise. https://trufflesecurity.com/trufflehog-enterprise
[^4]: Truffle Security, "Driftwood: Know if Private Keys are Sensitive." https://trufflesecurity.com/blog/driftwood
[^5]: Truffle Security, "Anyone can Access Deleted and Private Repo Data on GitHub." https://trufflesecurity.com/blog/anyone-can-access-deleted-and-private-repo-data-github
[^6]: TruffleHog README, local git scanning and `--clone-path` (CVE-2025-41390). https://github.com/trufflesecurity/trufflehog#scan-a-local-git-repo

## Tags

go, security, secret-scanning, credentials, devsecops, cli, static-analysis, git, ci-cd, secrets-detection, agpl
