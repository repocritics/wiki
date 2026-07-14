# aquasecurity/trivy

> A single-binary security scanner that finds CVEs, misconfigurations, secrets, and SBOM data across containers, filesystems, IaC, and Kubernetes.

[GitHub repo](https://github.com/aquasecurity/trivy) ·
[Official website](https://trivy.dev) ·
[License: Apache-2.0](https://github.com/aquasecurity/trivy/blob/main/LICENSE)

## Overview

Trivy is a Go security scanner from Aqua Security. It was started by Teppei Fukuda (knqyf263) and open-sourced in 2019 as a container-image vulnerability scanner that was easier to run than the incumbents (Clair, Anchore) — a single static binary, no server required, and a bundled vulnerability database[^1]. Aqua adopted the project shortly after, and over the following years its scope widened well past containers.

Today Trivy scans several *target* types (container image, filesystem, git repo, VM image, Kubernetes cluster, cloud account) with several *scanners* (OS-package and language-dependency vulnerabilities, IaC misconfigurations, hardcoded secrets, and software licenses), and can both generate and consume SBOMs in CycloneDX and SPDX[^2]. Much of that breadth came from absorbing other tools: the IaC misconfiguration engine is the former tfsec/defsec project folded in-tree[^3], and the Kubernetes operator descends from Starboard.

The defining tension is scope versus depth. Trivy's pitch is "one tool, one command" for an entire class of scanning that teams used to assemble from four or five separate projects. That breadth is real and is why it is the default scanner in countless CI pipelines. The cost is that no single scanner inside it is best-in-class, the tool has never declared a 1.0 (it remains on 0.x versioning after seven years and 70+ minor releases), and its correctness depends heavily on a large vulnerability database that must be downloaded and kept fresh.

## Getting Started

```bash
brew install trivy
# or: docker run aquasec/trivy
# or: download a binary from the releases page
```

```bash
# Scan a container image for OS + language CVEs
trivy image python:3.12-alpine

# Scan a project directory for vulns, secrets, and misconfigurations
trivy fs --scanners vuln,secret,misconfig ./myproject

# Fail CI only on HIGH/CRITICAL findings
trivy image --exit-code 1 --severity HIGH,CRITICAL myorg/myapp:latest
```

By default `trivy image` downloads the vulnerability database on first run (cached under `~/.cache/trivy`), analyzes each image layer, and prints a table of findings. `--format json`, `--format sarif`, and `--format cyclonedx` are the common machine-readable outputs.

## Architecture / How It Works

Trivy is built from a few internal libraries that map onto the target/scanner split:

- **Layer analysis (`fanal`).** For container and filesystem targets, Trivy walks the source (image layers, a directory tree, a tar) and runs a set of analyzers that recognize OS package databases (apk, dpkg, rpm) and language lockfiles/manifests (`package-lock.json`, `go.mod`, `pom.xml`, `requirements.txt`, `Gemfile.lock`, `Cargo.lock`, and many more). The output is a normalized list of installed packages with versions. Results are content-hashed and cached per layer, so re-scanning a similar image is fast.
- **Vulnerability matching (`trivy-db`).** The package list is matched against `trivy-db`, a bundled database compiled from many upstream advisory sources — NVD, per-distro security trackers (Red Hat, Debian, Alpine secdb, Amazon, SUSE, etc.), and GitHub Security Advisories[^4]. The DB is built server-side on a schedule and distributed as an OCI artifact from a container registry (GHCR by default), not as part of the binary. Java gets a separate `trivy-java-db` for offline JAR matching.
- **Misconfiguration scanning.** IaC files (Terraform, CloudFormation, Kubernetes YAML, Dockerfile, Helm, and others) are parsed into a common model and evaluated against policies written in Rego/OPA, shipped as the `trivy-checks` bundle (the successor to tfsec's built-in rules)[^3].
- **Secret scanning.** A built-in set of regex rules flags likely credentials (cloud keys, tokens, private keys) and is enabled by default for `fs` and `repo` scans.

Trivy can run standalone or in **client/server mode**: a long-running `trivy server` holds the vulnerability DB, and thin clients send package lists to it. This is the standard way to avoid every CI runner downloading the DB independently.

## Production Notes

**Database distribution is the main operational hazard.** The vulnerability DB is pulled from a container registry (GHCR) at runtime. Anonymous pulls in CI frequently hit registry rate limits, producing flaky, non-deterministic scan failures that have nothing to do with your code[^5]. The standard mitigations: mirror the DB to your own registry and point `--db-repository`/`TRIVY_DB_REPOSITORY` at it, authenticate the pull, persist `~/.cache/trivy` between CI runs, or use `--download-db-only` in a warm-up step. For air-gapped environments, bundle the DB with `trivy image --download-db-only` and ship it, then scan with `--skip-db-update`.

**No SemVer stability contract.** Trivy is still 0.x, and behavior-affecting changes — new default scanners, output-schema tweaks, matching-logic corrections — do land in minor releases. Pin the exact version in CI rather than tracking `latest`, and read release notes before bumping.

**Findings need triage tooling.** On real images the raw output is noisy: language-package CVEs with no fixed version, base-image vulns you cannot patch, distro CVEs the vendor has marked will-not-fix. Suppression is done with `.trivyignore`/`.trivyignore.yaml` files or, more auditably, with VEX documents that record *why* a finding does not apply. Budget for this — an unfiltered Trivy gate quickly gets ignored by developers.

**Caching and cold starts.** First run is dominated by the DB download (hundreds of MB decompressed). Steady-state scans are fast, but forget to persist the cache in CI and every job pays the download cost again. Large SBOM/monorepo scans can also spike memory.

**Exit codes are the CI contract.** `--exit-code` combined with `--severity` (and optionally `--ignore-unfixed`) is how you turn a report into a pass/fail gate. Without `--ignore-unfixed`, un-fixable base-image CVEs will fail builds you cannot actually fix.

## When to Use / When Not

**Use when:**
- You want one tool covering container, IaC, secret, and dependency scanning in CI instead of stitching several together.
- You need a fast, no-server, single-binary scanner that runs the same locally and in a pipeline.
- You want SBOM generation and consumption (CycloneDX/SPDX) alongside vuln scanning.

**Avoid (or supplement) when:**
- You need best-in-class depth in one area — a dedicated IaC platform, a specialized SAST tool, or a deeper secret scanner will each beat Trivy's built-in version of that scanner.
- You cannot tolerate a runtime DB download and are unwilling to mirror/air-gap it.
- You require a vendor SLA and stable, versioned API guarantees — the OSS tool is 0.x, with the paid Aqua platform as the commercial path.

## Alternatives

- anchore/grype — container/dependency vuln scanner; pair with anchore/syft for an SBOM-first workflow where SBOM generation and scanning are separate steps.
- quay/clair — registry-integrated scanner; use when you want scanning built into a registry backend (e.g. Quay) rather than a CLI.
- google/osv-scanner — OSV-database dependency scanner; use when you only care about open-source dependency CVEs sourced from OSV and want lockfile-focused output.
- bridgecrewio/checkov — IaC-only misconfiguration scanner with a large Python-based policy set; use when IaC coverage and policy depth matter more than breadth.
- trufflesecurity/trufflehog — dedicated secret scanner with credential verification; use when secret detection with active verification is the primary goal.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2019-05 | Initial open-source release: container-image vulnerability scanning[^1]. |
| — | 2019 | Adopted by Aqua Security. |
| 0.20 era | 2021 | IaC misconfiguration scanning added (via defsec); Starboard/operator lineage. |
| 0.27 | 2022 | Built-in secret scanning[^6]. |
| 0.x | 2022–2023 | SBOM generation/scan (CycloneDX, SPDX), Kubernetes and cloud/AWS targets, VEX support, Java DB. |
| 0.72.0 | 2026-06-30 | Current release at time of writing; still pre-1.0[^7]. |

## References

[^1]: Trivy — original announcement / project history. https://github.com/aquasecurity/trivy
[^2]: Trivy documentation, scanning coverage. https://trivy.dev/docs/latest/coverage/
[^3]: tfsec is now part of Trivy; the misconfiguration engine and Rego checks descend from tfsec/defsec. https://github.com/aquasecurity/tfsec
[^4]: trivy-db — vulnerability database sources and build. https://github.com/aquasecurity/trivy-db
[^5]: Trivy docs, "Air-Gapped Environment" and DB configuration (registry rate-limit mitigations). https://trivy.dev/docs/latest/advanced/air-gap/
[^6]: Trivy docs, secret scanning. https://trivy.dev/docs/latest/scanner/secret/
[^7]: Trivy releases. https://github.com/aquasecurity/trivy/releases

## Tags

go, security, vulnerability-scanner, container-security, sbom, iac, kubernetes, devsecops, secret-scanning, cli, static-analysis
