# goharbor/harbor

> A self-hosted, OCI-compliant container registry with RBAC, vulnerability scanning, replication, and signing bolted onto the upstream Distribution registry.

[GitHub repo](https://github.com/goharbor/harbor) ·
[Official website](https://goharbor.io) ·
[License: Apache-2.0](https://github.com/goharbor/harbor/blob/main/LICENSE)

## Overview

Harbor is a container/OCI-artifact registry that wraps the upstream Distribution
registry (formerly Docker Distribution) with the features enterprises actually
demand: projects with role-based access control, LDAP/AD and OIDC auth, image
vulnerability scanning, cross-registry replication, quotas, tag retention, and
signing. It began as an internal VMware project, was open-sourced in 2016, donated
to the CNCF in 2018, and graduated in 2020 — the first CNCF project to originate
from China to reach graduated status[^1]. As of 2026 it has ~28.9k stars and is one
of the default answers to "we need a private registry we control."

The defining tension is that Harbor is not a registry so much as a *distribution of
one*. The actual blob storage and OCI protocol are handled by the embedded
Distribution registry; Harbor is the control plane (core API, portal, job service,
scanner adapters, database, cache) around it. This buys a large feature surface but
also means Harbor is a multi-container system with stateful dependencies (PostgreSQL,
Redis, object storage), not a single binary. Operators trade the simplicity of a
plain registry for governance features, and inherit the operational weight that
comes with them.

## Getting Started

Harbor is deployed from an offline/online installer (Docker Compose) or a Helm
chart — not `go install`. Use tagged releases; `main` is explicitly documented as
possibly broken[^2].

```bash
# Docker Compose install on a Linux host (docker 20.10.10-ce+, compose 1.18.0+)
wget https://github.com/goharbor/harbor/releases/download/v2.15.0/harbor-offline-installer-v2.15.0.tgz
tar xzvf harbor-offline-installer-v2.15.0.tgz && cd harbor
cp harbor.yml.tmpl harbor.yml
# edit harbor.yml: set hostname, https certs, harbor_admin_password, storage
sudo ./install.sh --with-trivy      # enable the Trivy scanner
```

```bash
# push an image into a Harbor project
docker login core.harbor.example.com          # project must exist first
docker tag myapp:1.0 core.harbor.example.com/library/myapp:1.0
docker push core.harbor.example.com/library/myapp:1.0
```

For Kubernetes, use the separate `goharbor/harbor-helm` chart or the Harbor Operator
rather than running Compose in-cluster.

## Architecture / How It Works

Harbor is a set of cooperating containers, not a monolith:

- **harbor-core** — the API server and business logic (Go). Owns projects, RBAC,
  robot accounts, quotas, retention/immutability policies, and orchestrates the
  other services.
- **registry** — the upstream Distribution registry, doing the actual OCI push/pull
  and blob storage. Harbor proxies and authorizes requests to it via a token service.
- **harbor-portal** — the Angular single-page UI.
- **jobservice** — an async worker pool (backed by Redis) for replication, GC,
  scan-all, retention, and other long-running jobs.
- **trivy-adapter** — the default vulnerability scanner since 2.2 (Aqua's Trivy,
  which replaced Clair)[^3]. Scanning is pluggable via the scanner-adapter API.
- **PostgreSQL** — metadata store (the sole supported RDBMS; MySQL was dropped).
- **Redis** — caching and the job queue.

Because storage is delegated to Distribution, Harbor inherits its storage-driver
support: local filesystem, S3, GCS, Azure Blob, Swift, and others. Signing moved
from Notary v1 (Docker Content Trust) toward Cosign/sigstore verification; release
artifacts themselves are Cosign-signed as of v2.15[^2]. Replication is
adapter-based, with pull/push connectors for Docker Hub, GCR, ECR, ACR, Quay, GitLab,
and more. Optional add-ons include proxy (pull-through) cache projects and P2P
preheat via Dragonfly or Kraken.

## Production Notes

**Garbage collection is the classic footgun.** Reclaiming space from deleted tags
requires a GC run, and historically GC put the registry into read-only mode, blocking
pushes for its duration. Non-blocking GC has improved, but GC on a large registry is
still I/O-heavy and long-running; schedule it in low-traffic windows and expect it to
touch every blob[^4].

**It is stateful and HA is non-trivial.** The default Compose install is single-node.
Real high availability means external HA PostgreSQL, external HA Redis, and shared
object storage (S3/GCS) so multiple Harbor cores are stateless in front of them.
There is no built-in database clustering — that is on you.

**Scanning has its own operational cost.** Trivy needs a vulnerability DB that must be
refreshed (internet egress or an offline DB mirror in air-gapped setups), and
"scan all" across a big registry is resource-intensive. CVE results are only as fresh
as the DB and only cover what the scanner understands.

**Upgrades are migrator-gated.** Harbor runs schema migrations on upgrade and expects
you to move one minor version at a time rather than skipping across several; always
back up the PostgreSQL database and `harbor.yml` before upgrading, and read the
release notes for breaking config changes[^5].

**Deprecations to watch.** ChartMuseum-based Helm chart hosting was deprecated in
favor of storing Helm charts as OCI artifacts; Notary v1 signing was deprecated. Both
have bitten operators who built workflows on the old paths.

## When to Use / When Not

**Use when:**
- You need a self-hosted, on-prem, or air-gapped registry with real RBAC, LDAP/OIDC,
  and audit logging.
- You want built-in vulnerability scanning and policy gates on image deployment.
- You need to replicate images across regions/clouds or mirror upstream registries.
- You want a CNCF-graduated project with an established governance and security track.

**Avoid when:**
- You just need a plain registry — Distribution or zot is far lighter to run.
- You can use a managed registry (ECR/GAR/ACR/GHCR) and don't need on-prem control.
- You lack the operational capacity to run HA PostgreSQL, Redis, and object storage.
- You want a single static binary; Harbor is a multi-service system by design.

## Alternatives

- distribution/distribution — the upstream registry Harbor is built on; use it when you want raw OCI push/pull with no UI, RBAC, or scanning.
- project-zot/zot — lightweight OCI-native registry; use it when you want a small, single-binary registry with optional UI and scanning.
- quay/quay — Red Hat's registry with similar scanning/RBAC; use it when you're in the Red Hat/OpenShift ecosystem.
- AWS ECR / Google Artifact Registry / Azure ACR — use a managed cloud registry when you don't need self-hosting and want zero operational burden.
- jfrog/artifactory or Sonatype Nexus — use these when you need one artifact store spanning containers plus Maven, npm, PyPI, etc.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-03 | Open-sourced by VMware as Project Harbor[^1]. |
| 1.0 | 2017 | First GA; projects, RBAC, LDAP, replication. |
| — | 2018-07 | Accepted into the CNCF Sandbox[^6]. |
| 2.0 | 2020-05 | OCI-compliant rewrite; OCI artifacts and Helm-as-OCI[^7]. |
| — | 2020-06 | Graduated in the CNCF[^1]. |
| 2.2 | 2021 | Trivy becomes the default scanner (replacing Clair)[^3]. |
| 2.15 | 2025 | Cosign-signed release artifacts (verifiable via sigstore)[^2]. |

## References

[^1]: CNCF, "Harbor" project page and graduation announcement. https://www.cncf.io/projects/harbor/
[^2]: Harbor README — release/branch guidance and Cosign signature verification. https://github.com/goharbor/harbor/blob/main/README.md
[^3]: Harbor docs, "Vulnerability Scanning" (Trivy default scanner). https://goharbor.io/docs/latest/administration/vulnerability-scanning/
[^4]: Harbor docs, "Garbage Collection." https://goharbor.io/docs/latest/administration/garbage-collection/
[^5]: Harbor docs, "Upgrade Harbor." https://goharbor.io/docs/latest/administration/upgrade/
[^6]: CNCF blog, "CNCF to Host Harbor in the Sandbox" — 2018-07-31. https://www.cncf.io/blog/2018/07/31/cncf-to-host-harbor-in-the-sandbox/
[^7]: Harbor blog, "Harbor 2.0" (OCI support). https://goharbor.io/blog/harbor-2.0/

## Tags

go, container-registry, oci, cloud-native, cncf, kubernetes, docker, vulnerability-scanning, rbac, self-hosted, devops, image-replication
