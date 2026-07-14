# anchore/syft

> A CLI and Go library that generates a Software Bill of Materials (SBOM) by cataloging packages in container images and filesystems.

[GitHub repo](https://github.com/anchore/syft) ·
[Docs](https://oss.anchore.com/docs/) ·
[License: Apache-2.0](https://github.com/anchore/syft/blob/main/LICENSE)

## Overview

Syft is Anchore's SBOM generator: point it at a container image, a directory, or an
archive and it enumerates the software packages it can find — OS packages (apk, dpkg,
RPM) and language packages (Go, Python, Java, JavaScript, Ruby, Rust, PHP, .NET, and
more) — then emits that inventory in a standard format such as CycloneDX, SPDX, or its
own Syft JSON[^1]. It does not scan for vulnerabilities itself; that is the job of its
sibling tool Grype, which consumes a Syft SBOM as input[^2]. The two are developed in
lockstep and the split is deliberate: cataloging (what is installed) is separated from
matching (what is vulnerable).

The audience is anyone who needs a machine-readable inventory of what shipped: security
teams answering "are we affected by CVE-X," compliance functions producing SBOMs for
regulations like the US Executive Order 14028 / NTIA minimum-elements guidance, and CI
pipelines attaching an SBOM to every build artifact. Syft is one of the two or three
tools (alongside Trivy and the CycloneDX generators) that most SBOM tooling in the wild
actually runs.

The defining tension is accuracy versus reach. Syft is a *cataloger*, not a package
manager or build system, so it reconstructs the package list from evidence left on disk
— manifest files, package databases, and metadata embedded in compiled binaries. That
makes it work almost anywhere without a build environment, but it also means results are
heuristic: a stripped Go binary, a vendored dependency with no manifest, or an unusual
install layout can be under- or over-reported. An SBOM from Syft is a strong best-effort
inventory, not a cryptographic proof of contents.

## Getting Started

```bash
# install (or use Homebrew, Docker, Scoop, Chocolatey, Nix — see docs)
curl -sSfL https://get.anchore.io/syft | sudo sh -s -- -b /usr/local/bin
```

```bash
# list packages in an image or directory (human-readable table)
syft alpine:latest
syft ./my-project

# emit an SBOM in a standard format
syft alpine:latest -o cyclonedx-json

# write multiple formats at once
syft alpine:latest -o spdx-json=./spdx.json -o cyclonedx-json=./cdx.json
```

```bash
# feed the SBOM to Grype for vulnerability matching (no re-scan needed)
syft alpine:latest -o syft-json=sbom.json
grype sbom:./sbom.json
```

## Architecture / How It Works

Syft's pipeline has three stages: **source → catalog → encode**.

- **Source.** An input is resolved to a uniform file tree. Container images are pulled
  and unpacked via `stereoscope`, Anchore's image library, which understands OCI and
  Docker layer formats and can present either the squashed final filesystem or all
  layers[^3]. Directories and archives are resolved directly. Everything downstream sees
  the same `Resolver` abstraction, so catalogers do not care whether they are reading an
  image or a folder.
- **Catalog.** A set of ecosystem-specific *catalogers* walks the resolved tree. Each
  cataloger knows how to recognize and parse one kind of evidence — `dpkg` status files,
  RPM databases, `package-lock.json`, `go.mod` plus Go build metadata embedded in
  binaries, Python `*.dist-info`, Java `MANIFEST.MF` inside JARs, and so on. Catalogers
  run in parallel and each yields `Package` records plus `Relationship` edges (for
  example "contained-by" or "dependency-of").
- **Encode.** The collected packages, relationships, and source description form an
  internal SBOM model. Format encoders serialize that model to CycloneDX, SPDX (JSON or
  tag-value), Syft JSON, and others. Syft JSON is the lossless native format; SPDX and
  CycloneDX are standardized but lossy — some Syft-specific fields have no home in them.

The cataloger set is the heart of the project and where nearly all contributions land.
Coverage is uneven by design: mature ecosystems (Debian, RPM, npm, Go, Java) are solid;
newer or niche ones are best-effort. Because catalogers evolve release to release, the
same image scanned with two different Syft versions can produce meaningfully different
SBOMs — an important property to understand before you treat SBOMs as stable baselines.

## Production Notes

- **Syft finds packages, not vulnerabilities.** There is no CVE data inside Syft. Pair
  it with Grype (or another matcher that reads Syft JSON) — running `grype` against a
  pre-generated SBOM avoids re-cataloging and is the recommended CI pattern.
- **Pin the Syft version for reproducibility.** Cataloger behavior changes between
  releases, so an unpinned `latest` in CI will silently shift SBOM contents over time.
  For diffable, auditable SBOMs, pin an exact version and bump deliberately.
- **Format choice is lossy and versioned.** CycloneDX and SPDX each have multiple spec
  versions; the output flag encodes both format and version (e.g. `spdx-json`,
  `spdx-tag-value`, `cyclonedx-xml`). Downstream tools are often picky about which spec
  version they accept — validate against your consumer, not just against the schema.
- **Binary detection is heuristic.** Go binaries are cataloged from embedded build info;
  stripping or building without module info loses that data. Statically linked C/C++ and
  vendored code without manifests are frequently missed — Syft reports what it can prove,
  and absence of a package is not proof of absence.
- **Image scope matters.** By default Syft catalogs the squashed image; `--scope
  all-layers` also inspects files deleted in later layers (useful for finding secrets or
  packages removed at build time, but noisier and slower).
- **Relationships are partial.** Syft records package relationships where the evidence
  supports them, but it is not a full dependency resolver — do not expect a complete,
  transitive dependency graph from every ecosystem.
- **Attestations.** Syft can produce in-toto attestations for signing SBOMs (typically
  via cosign), fitting supply-chain provenance workflows — but key management and policy
  live outside Syft.

## When to Use / When Not

**Use when:**
- You need SBOMs for images or source trees in CI and want broad ecosystem coverage.
- You already use Grype, or want a clean cataloging/matching split.
- You must emit standard CycloneDX or SPDX for compliance or downstream tooling.
- You want a Go library (`github.com/anchore/syft/syft`) to embed cataloging in your own tool.

**Avoid / look elsewhere when:**
- You want vulnerability results directly — that is Grype/Trivy, not Syft.
- You need a guaranteed-complete inventory — build-time SBOMs from the package manager or
  build system are more authoritative than after-the-fact filesystem cataloging.
- Your stack is dominated by stripped static binaries or exotic packaging Syft doesn't catalog.
- You want a single all-in-one scanner UX — Trivy bundles cataloging and scanning together.

## Alternatives

- aquasecurity/trivy — use instead when you want one tool that both generates SBOMs and scans for vulnerabilities, misconfigs, and secrets.
- anchore/grype — the companion, not a competitor: use for the vulnerability-matching half that Syft deliberately omits.
- CycloneDX/cyclonedx-cli — use when you need to convert, merge, or validate CycloneDX documents rather than generate them.
- microsoft/sbom-tool — use when you want SPDX SBOMs tied into a Microsoft/Azure build-provenance workflow.
- kubernetes-sigs/bom — use for SPDX generation in Kubernetes-centric release pipelines.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-05 | Repository created; Syft launched as Anchore's standalone SBOM CLI[^4]. |
| 0.x | 2020–2023 | Rapid cataloger expansion, added SPDX and CycloneDX encoders, attestation support. |
| 1.0.0 | 2024-02 | First stable release with API/format compatibility guarantees[^5]. |
| 1.x | 2024–2026 | Ongoing cataloger coverage, format-version updates, and library-API refinements. |

## References

[^1]: Syft README — features and supported formats. https://github.com/anchore/syft
[^2]: Grype — vulnerability scanner that consumes Syft SBOMs. https://github.com/anchore/grype
[^3]: stereoscope — container image/layer library used by Syft. https://github.com/anchore/stereoscope
[^4]: anchore/syft repository metadata (created 2020-05-07). https://github.com/anchore/syft
[^5]: Anchore blog — "Syft reaches v1.0.0" (Feb 2024). https://anchore.com/blog/

## Tags

sbom, supply-chain-security, go, cli, container-security, cyclonedx, spdx, static-analysis, devsecops, anchore, oci, dependency-scanning
