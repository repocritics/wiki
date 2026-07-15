# distribution/distribution

> The reference implementation of the OCI Distribution Spec — the registry server (`registry:2`) that Docker Hub, GHCR, GitLab, and Harbor are built on top of.

[GitHub repo](https://github.com/distribution/distribution) ·
[Official website](https://distribution.github.io/distribution) ·
[License: Apache-2.0](https://github.com/distribution/distribution/blob/main/LICENSE)

## Overview

`distribution` is the open-source registry that stores and serves container images and other OCI artifacts over HTTP. It began as the Registry v2 rewrite inside Docker (`docker/distribution`), replacing the Python-based v1 registry with a Go implementation and a content-addressable storage model[^1]. The project was donated to the CNCF and moved to the neutral `distribution/distribution` org, where it now serves as the reference implementation of the OCI Distribution Specification[^2].

The defining fact about this repo is that it is a *library and a base*, not a product. It ships as a single Go binary (published as the `registry:2` Docker image) that speaks the pull/push protocol correctly and stores blobs, but it deliberately omits almost everything a hosted registry needs: no web UI, no user management, no image scanning, no replication, no quota system, and only rudimentary authentication. Docker Hub, GitHub Container Registry, GitLab Container Registry, DigitalOcean, and the CNCF Harbor project all wrap this codebase with those missing layers[^3]. So the project's tension is that it is simultaneously the most widely deployed registry core in existence and something you rarely run raw in production without building around it.

A second caveat the maintainers state plainly: the Go library interfaces (the `registry/client` package and storage driver APIs) are marked **unstable**, and the bundled client is deprecated in favor of containerd's `remotes/docker` implementation[^4]. Consuming this repo as a Go dependency is not the supported path; consuming it as a running registry server is.

## Getting Started

Run a local registry with the filesystem driver:

```bash
docker run -d -p 5000:5000 --name registry \
  -v /mnt/registry:/var/lib/registry \
  registry:2
```

Push and pull against it:

```bash
docker pull alpine:latest
docker tag alpine:latest localhost:5000/alpine:latest
docker push localhost:5000/alpine:latest
docker pull localhost:5000/alpine:latest
```

Configuration is a single YAML file (`/etc/docker/registry/config.yml`), mounted in to override the defaults — storage backend, auth, notifications, and HTTP TLS all live here:

```yaml
version: 0.1
storage:
  s3:
    region: us-east-1
    bucket: my-registry
  delete:
    enabled: true        # required before any blob deletion / GC
http:
  addr: :5000
  secret: a-random-string # signs state in pagination / upload tokens
```

## Architecture / How It Works

The server exposes the OCI Distribution HTTP API — a small set of routes under `/v2/` for manifests, blobs, blob uploads (chunked and monolithic), and tag listing. Everything is **content-addressed**: blobs and manifests are keyed by their SHA-256 digest, and a tag is just a mutable pointer to a manifest digest. The same layer pushed under ten repositories is stored once.

Storage is abstracted behind a **storage driver** interface. The historically supported drivers are `filesystem`, `s3`, `azure`, `gcs`, `swift`, `oss`, and `inmemory` (test-only)[^5]. Everything above the driver — the blob store, manifest service, tag store, and upload session handling — is driver-agnostic and treats the backend as a flat key/value blob namespace. This is why S3 and a local disk behave identically at the API layer, and also why some backends (object stores without real directories) have quirks around listing and atomic renames.

Authentication is intentionally minimal. The built-in options are `silly` (dev only, trusts any request), `htpasswd` (basic auth against a bcrypt file), and `token`. The `token` scheme is the real one: the registry does not authenticate users itself — it redirects clients to an **external authorization server** that issues signed JWTs scoped to `repository:name:pull,push`, and the registry only verifies the signature and scope[^6]. Docker Hub, Harbor, and GitLab each run their own token server. There is no user database in this repo.

Two other subsystems matter operationally: **notifications**, which POST webhook events to configured endpoints on push/pull/delete (used to trigger downstream scanning or replication), and **garbage collection**, a separate `registry garbage-collect` command that mark-and-sweeps unreferenced blobs. GC is not a background process — it is a manual, offline batch job (see below).

## Production Notes

**Garbage collection is the classic footgun.** Deleting a tag or manifest only removes the reference; the underlying blobs stay on disk until GC runs. GC is a two-phase mark-and-sweep, and it is **not safe to run against a writable registry** — a push in flight can upload a layer that GC has already decided is unreferenced, and GC will delete it. The supported procedure is to put the registry in read-only mode (`storage.maintenance.readonly.enabled: true`) or stop it entirely, then run `registry garbage-collect config.yml`[^7]. On large registries this is a real availability cost and the single most common operational complaint.

**Deletes are digest-only and disabled by default.** You cannot `DELETE` by tag through the API; you resolve the tag to a manifest digest and delete that. Deletion must be enabled with `storage.delete.enabled: true` or the endpoint returns 405. Untagging does not free space (see GC above).

**S3 is the standard production backend, with caveats.** Multipart upload sizing, eventual-consistency edge cases on some S3-compatible stores, and the cost of `LIST` operations during GC and catalog enumeration all show up at scale. The `_catalog` endpoint and tag listing walk the storage tree and are slow/expensive on registries with many repositories — they are paginated but not indexed.

**No HA state, but stateless replicas work.** The server holds no local state beyond the storage backend and in-flight upload sessions, so you can run multiple replicas behind a load balancer pointing at the same S3 bucket. Chunked-upload sessions are the exception: an in-progress blob upload is tied to the instance that started it unless you configure a shared `redis` cache and sticky routing.

**Pull-through cache mode** turns the registry into a mirror of an upstream (e.g. Docker Hub) to cut egress and rate-limit exposure. It caches blobs on first pull. It cannot be used as a push target at the same time.

**The `registry:2` tag is a moving target.** It tracks the 2.x line; the 3.0 release modernized the codebase (Go module path, updated OCI spec support, dependency cleanup) and changed some defaults and driver packaging[^8]. Read the release notes before bumping a long-lived deployment — config keys and bundled storage drivers have shifted between major lines.

## When to Use / When Not

**Use when:**
- You need a spec-correct private registry and are fine running it raw or lightly wrapped.
- You want a pull-through cache/mirror in front of a public registry.
- You are building a registry product and want a proven OCI-compliant core to extend (the Harbor/GitLab pattern).
- You want a single static binary with pluggable object-store backends and no external database.

**Avoid when:**
- You need a UI, user/team management, vulnerability scanning, quotas, or replication out of the box — reach for Harbor or a hosted registry instead.
- You want turnkey authentication; the token flow requires you to run a separate auth server.
- You depend on the Go client library as a stable API surface — it is explicitly unstable and deprecated.
- You want online, zero-downtime garbage collection — this design requires read-only/offline GC.

## Alternatives

- goharbor/harbor — full registry *product* built on this codebase; adds UI, RBAC, scanning, replication, quotas. Use when you need a batteries-included private registry.
- project-zot/zot — OCI-native registry with no Docker daemon lineage, built-in UI and search, optional scanning. Use when you want a modern, spec-first server with more features than raw distribution.
- Quay (quay/quay) — Red Hat's registry with scanning and mirroring. Use when you're in the Red Hat/OpenShift ecosystem.
- Sonatype Nexus / JFrog Artifactory — multi-format artifact managers that also do OCI. Use when containers are one of many artifact types you host.
- Cloud managed registries (ECR, GAR, ACR) — use when you don't want to operate storage, GC, or availability yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| v2.0 | 2016-04 | Registry v2 GA; content-addressable Go rewrite of the v1 Python registry[^1]. |
| v2.6 | 2017-01 | Manifest list (multi-arch) support maturing. |
| v2.7 | 2019-01 | Last major 2.x feature line under `docker/`; broad storage-driver support. |
| — | 2019 | Project donated to the CNCF; repo moved to `distribution/distribution`[^2]. |
| v2.8 | 2021-08 | OCI image-spec media types, maintenance under CNCF governance. |
| v3.0 | 2025 | First major release under CNCF; Go module path change, updated OCI Distribution Spec support, dependency and driver cleanup[^8]. |

## References

[^1]: Docker, "The next generation Docker Registry" — announcing Registry v2 / distribution. https://www.docker.com/blog/registry-v2-2-0/
[^2]: CNCF, "Distribution project" (moved from `docker/distribution` to CNCF). https://www.cncf.io/projects/distribution/
[^3]: distribution README — list of registry operators built on the codebase (Docker Hub, GHCR, GitLab, Harbor). https://github.com/distribution/distribution
[^4]: distribution README — "the interfaces for these libraries are unstable"; client deprecated in favor of containerd `remotes/docker`. https://github.com/containerd/containerd/tree/main/core/remotes/docker
[^5]: distribution docs, "Storage drivers". https://distribution.github.io/distribution/storage-drivers/
[^6]: distribution docs, "Token authentication specification". https://distribution.github.io/distribution/spec/auth/token/
[^7]: distribution docs, "Garbage collection" — run in read-only mode. https://distribution.github.io/distribution/about/garbage-collection/
[^8]: distribution releases (3.0 line). https://github.com/distribution/distribution/releases

## Tags

go, container-registry, oci, docker, image-registry, cncf, object-storage, devops, self-hosted, infrastructure
