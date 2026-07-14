# sigstore/cosign

> Container and artifact signing built around short-lived certificates and a public transparency log, so signatures need no long-lived keys.

[GitHub repo](https://github.com/sigstore/cosign) ·
[Sigstore project](https://sigstore.dev) ·
[License: Apache-2.0](https://github.com/sigstore/cosign/blob/main/LICENSE)

## Overview

Cosign is the signing and verification CLI of the Sigstore project, first released in 2021 and now a mature, widely deployed tool (~6.1k stars) rather than a fast-growing one[^1]. Its purpose is narrow: sign OCI container images and arbitrary blobs, and verify those signatures. What made it notable is the *keyless* model — instead of managing a long-lived private key, you authenticate with an OIDC identity (a Google/GitHub/Microsoft account, or a CI workload token), and Sigstore's Fulcio CA issues a short-lived (~10 minute) X.509 code-signing certificate bound to that identity[^2].

The defining tradeoff is **ephemeral keys plus a public transparency log**. Because the signing certificate expires in minutes, verification cannot simply check "is this cert still valid" — it relies on the Rekor transparency log to prove the signature was created while the certificate was live[^3]. This removes key-management burden but pushes your signing identity (including the email address you authenticated with) into a permanent, public, append-only log that cannot be redacted. That is an explicit, non-negotiable property, and cosign prompts you to consent to it on every keyless sign.

Cosign 2.x is the current stable line. The maintainers have publicly stated that new feature development is shifting to the [sigstore-go](https://github.com/sigstore/sigstore-go) library, with cosign 2.x receiving periodic fixes and small features but no breaking API changes[^1]. Treat it as a stable, slowing-down tool, not a fast-moving one.

## Getting Started

```shell
# Homebrew / Arch / Nix / GitHub Action installers, or grab a release binary:
brew install cosign
# or build from source (Go 1.22+):
go install github.com/sigstore/cosign/v2/cmd/cosign@latest
```

Sign a container by digest (never by tag — a tag can move under you):

```shell
# Keyless: opens a browser for OIDC, issues a Fulcio cert, logs to Rekor.
cosign sign $IMAGE@sha256:...

# Verify — identity and issuer are MANDATORY for keyless (see History).
cosign verify $IMAGE@sha256:... \
  --certificate-identity=me@example.com \
  --certificate-oidc-issuer=https://accounts.google.com
```

Sign a blob and verify against a CI workload identity:

```shell
cosign sign-blob artifact --bundle artifact.sigstore.json --yes
cosign verify-blob artifact --bundle artifact.sigstore.json \
  --certificate-identity "https://github.com/ORG/REPO/.github/workflows/release.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com"
```

## Architecture / How It Works

Cosign is a client that orchestrates four Sigstore components, none of which it implements itself:

- **Fulcio** — a certificate authority that exchanges an OIDC identity token for a short-lived code-signing certificate. The certificate's subject is your OIDC identity; there is no enrollment or long-lived key.
- **Rekor** — an append-only transparency log (a Merkle tree). Every keyless signature is uploaded here, producing a log entry with an inclusion proof and an integrated timestamp. This is what makes short-lived certs verifiable after they expire.
- **TUF** — The Update Framework distributes the *trusted root* (`trusted_root.json`): the set of Fulcio/Rekor public keys clients must trust. `cosign initialize` fetches it. Its contents change without notice, which matters for air-gapped setups.
- **go-containerregistry** — handles all registry I/O, giving cosign broad registry compatibility.

For keys, cosign also supports the classic model: `cosign generate-key-pair` (or a KMS/HSM/PIV token), sign with `--key`, verify with `--key`. Generated keys are always ECDSA-P256 with SHA-256, stored as PEM-encoded PKCS8[^4]; KMS backends can use other algorithms.

**How signatures are stored.** A signature is a *separate* OCI object pushed alongside the image, addressed by a derived tag (`sha256-<digest>.sig`). The link from signature to image is only this weak naming convention — the registry itself has no idea the object is a signature. Attestations (in-toto/DSSE predicates) and SBOMs are stored the same way. Newer cosign versions emit a protobuf "bundle" (a self-contained sigstore bundle) that packages the signature, certificate, and Rekor proof for offline verification.

## Production Notes

**Signatures are never garbage-collected.** Because the signature is a loosely-referenced separate object, deleting an image does *not* delete its signatures — orphaned `.sig` objects accumulate in your registry indefinitely, and you must clean them up yourself[^4].

**Concurrent signing has a race.** Multiple signatures on one image are stored in a list updated via read-append-write; under contention the last write wins and an earlier signature can be silently dropped[^4]. Serialize signing of the same digest.

**You depend on Sigstore uptime and rate limits.** Keyless signing needs Fulcio, Rekor, *and* your OIDC provider all reachable at sign time. The public-good instance is best-effort and rate-limited; the README's own troubleshooting guidance for signing failures is "retry." For anything load-bearing, either use KMS keys (no Sigstore services on the signing path) or run your own Fulcio/Rekor/TUF stack.

**Privacy is permanent.** Keyless signing writes your authenticated email (or workload identity) into the public Rekor log forever. In CI this is fine (the identity is a workflow), but individuals signing with a personal email are publishing that email. Use a workload identity or a managed key if that is unacceptable.

**Air-gapped verification is fiddly.** Offline verify works only if the signature carries a bundle, and you must supply a `trusted_root.json` that you keep up to date out-of-band — the file changes without notification and there is no built-in sync for airgapped copies.

**Timestamps and log versions cause upgrade churn.** Recent errors like "threshold not met for verified log entry integrated timestamps" (RFC3161 timestamp support, `--use-signed-timestamps` in 2.6.x) or "no signatures found" against a Rekor v2 log are resolved by upgrading cosign — old clients cannot verify newer signing modes. Keep signer and verifier versions close.

**Policy enforcement is a separate layer.** Cosign verifies; it does not enforce. In Kubernetes, admission is done by sigstore's `policy-controller`, or by Kyverno / OPA Gatekeeper calling cosign verification logic. Do not confuse "images are signed" with "unsigned images are blocked."

## When to Use / When Not

**Use when:**
- You want to sign container images or release artifacts without operating key infrastructure.
- Your signing happens in CI, where a workflow OIDC identity is a natural, privacy-safe signer.
- You want public auditability (transparency log) as part of your supply-chain story (SLSA, in-toto attestations, SBOM attachment).
- You need broad registry compatibility and a single tool for images, blobs, and attestations.

**Avoid / reconsider when:**
- You cannot tolerate signing identities in a public log and cannot run a private Sigstore instance.
- You need a signing operation with no external service dependencies at sign time — use `--key` with a KMS instead of keyless.
- You are on cosign 1.x scripts — they will not survive the 2.x verification changes (see History); plan the migration rather than pinning forever.
- Your registry or environment is fully air-gapped and you cannot maintain the TUF trusted root manually.

## Alternatives

- notaryproject/notation — CNCF Notary v2 signing; X.509/PKI-based with no mandatory transparency log, closer to traditional enterprise CA workflows. Use when you want registry signing without a public log and already have a CA.
- sigstore/sigstore-go — the Go library where new Sigstore signing/verification features now land; embed this instead of shelling out to cosign, or when you need bring-your-own-key flows.
- slsa-framework/slsa-verifier — verifies SLSA provenance for releases; use alongside or instead when your goal is provenance verification, not raw signatures.
- in-toto/in-toto — supply-chain attestation framework; use when you need to attest a full build pipeline rather than sign a single artifact (cosign can carry in-toto predicates).
- gpg / skopeo simple-signing — classic detached GPG signatures for images; use only in environments already standardized on GPG and long-lived keys.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-02 | Repo created; project started by Dan Lorenc, part of Sigstore[^1]. |
| 1.0.0 | 2022-07 | First stable release; keyless signing via Fulcio/Rekor matured. |
| 2.0.0 | 2023-03 | Major release: `--certificate-identity` and `--certificate-oidc-issuer` became **mandatory** for keyless verify, closing a gap where any Fulcio-issued cert would pass[^2]. |
| 2.2–2.4 | 2023–2024 | Protobuf bundle format, trusted-root handling, KMS and registry improvements. |
| 2.6.x | 2025 | RFC3161 signed-timestamp verification (`--use-signed-timestamps`), Rekor v2 support. |

## References

[^1]: Cosign README and repository metadata (Apache-2.0, Go, created 2021-02-04). Maintainers note future development focus on sigstore-go. https://github.com/sigstore/cosign
[^2]: Sigstore documentation, keyless signing and Fulcio certificate model. https://docs.sigstore.dev/cosign/signing/overview/
[^3]: Rekor transparency log — inclusion proofs and integrated timestamps enable verification of short-lived certificates. https://docs.sigstore.dev/logging/overview/
[^4]: Cosign README, "Caveats": ECDSA-P256/SHA256 keys, simple-signing payloads, signatures stored as separate un-garbage-collected OCI objects, and the multi-signature read-append-write race. https://github.com/sigstore/cosign#caveats

## Tags

go, container-signing, supply-chain-security, sigstore, keyless-signing, transparency-log, oci, code-signing, cli, devsecops
