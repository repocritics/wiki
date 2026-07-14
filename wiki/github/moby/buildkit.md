# moby/buildkit

> The concurrent, content-addressable build engine that runs inside `docker build` — and the substrate under Buildx, Dagger, Earthly, and Depot.

[GitHub repo](https://github.com/moby/buildkit) ·
Docs: [docs.docker.com/build](https://docs.docker.com/build/) ·
[License: Apache-2.0](https://github.com/moby/buildkit/blob/master/LICENSE)

## Overview

BuildKit is a toolkit for turning source into build artifacts — most commonly OCI/Docker
images — in a way that is concurrent, aggressively cached, and not tied to the Dockerfile
language. It grew out of a 2017 proposal to replace the legacy image builder inside the
Docker Engine[^1] and has since become the default `docker build` backend as of Docker
Engine 23.0 (2023)[^2]. If you have run `docker build` on any recent Docker install, you
have used BuildKit whether you knew it or not.

The project ships two pieces: `buildkitd`, a long-lived daemon that owns the build cache
and executes work, and `buildctl`, a thin gRPC client. Most users never touch these
directly — they go through Buildx (the `docker build` / `docker buildx` CLI) which wraps
the daemon. The standalone `buildctl`/`buildkitd` pair is what you reach for when embedding
BuildKit in CI systems, Kubernetes, or higher-level tools. This split — a low-level engine
that almost everyone consumes through a wrapper — is the defining shape of the project and
the source of most of its documentation confusion: the README repeatedly tells casual
visitors they probably don't need to read it[^3].

Its central idea is **LLB** (Low-Level Build), a protobuf-encoded dependency graph that is
"to Dockerfile what LLVM IR is to C"[^3]. Dockerfiles are compiled to LLB by a *frontend*;
the solver then executes the LLB DAG concurrently with content-addressable caching. Because
LLB is vendor-neutral, non-Dockerfile languages (Earthfile, HLB, Nix, Buildpacks, Mockerfile)
target the same engine.

## Getting Started

```bash
# Daemon (Linux, needs root; runc or containerd present):
sudo buildkitd &

# macOS: client only via Homebrew; run the daemon in a Linux VM (Lima):
brew install buildkit lima
limactl start template://buildkit
export BUILDKIT_HOST="unix://$HOME/.lima/buildkit/sock/buildkitd.sock"
```

```bash
# Build a Dockerfile and push the resulting image:
buildctl build \
  --frontend=dockerfile.v0 \
  --local context=. \
  --local dockerfile=. \
  --output type=image,name=docker.io/youruser/app:latest,push=true \
  --export-cache type=inline \
  --import-cache type=registry,ref=docker.io/youruser/app:latest
```

`--local context=.` and `--local dockerfile=.` stream the source and the Dockerfile from the
client to the daemon; `--output` decides where the result goes (`image`, `local`, `tar`,
`docker`, `oci`, or the containerd image store). In practice most people write a Dockerfile
and let `docker build` invoke all of this for them.

## Architecture / How It Works

The pipeline is **frontend → LLB → solver → worker → exporter**.

- **Frontends** convert a build definition to LLB. `dockerfile.v0` is compiled into the repo
  today but is designed to move out into an external image ([#163])[^3]; `gateway.v0` lets any
  container image act as a frontend, which is how `# syntax=docker/dockerfile:1` pins a
  specific Dockerfile parser version pulled from a registry at build time. This is why new
  Dockerfile features (`RUN --mount`, heredocs, `COPY --link`) can ship independently of the
  Docker Engine release cycle.
- **LLB** is a Merkle DAG of operations (exec, file ops, source fetches). Every vertex is
  content-addressed, so identical subgraphs across builds share cache entries. This is a
  genuinely different model from the legacy builder's linear layer-per-instruction cache: it
  enables parallel execution of independent stages and fine-grained `RUN --mount=type=cache`
  mounts that persist package-manager caches across builds without baking them into layers.
- **Workers** execute the exec ops in sandboxed containers. Two backends exist: the default
  OCI worker (runc/crun) and a containerd worker (`--oci-worker=false --containerd-worker=true`).
- **Exporters** serialize the result; **cache exporters** (`inline`, `registry`, `local`,
  `gha`, `s3`, `azblob`) serialize the cache separately so CI runners can share it.

Later versions added build **attestations** — SLSA provenance and SBOMs attached to images as
OCI referrers — pushing BuildKit into the supply-chain-security story around v0.11[^4].
Multi-platform images are produced either by QEMU emulation or by cross-compilation inside a
single Dockerfile using the `TARGETPLATFORM`/`BUILDPLATFORM` args.

## Production Notes

- **The cache is stateful and lives with the daemon.** `buildkitd` owns `/var/lib/buildkit`.
  Garbage collection is on by default and policy-driven; misconfigured GC (or none) is the
  classic way to fill a CI host's disk. Inspect with `buildctl du -v`, reclaim with
  `buildctl prune`. There is no built-in cross-host shared local cache — sharing across
  ephemeral runners means exporting to a `registry`/`gha`/`s3` cache backend, and cold cache
  imports can themselves be slow for large layer sets.
- **Cache mode `min` vs `max` is a real footgun.** The convenient `inline` exporter only
  supports `min` (final-stage layers). Caching intermediate stages requires `max` mode with a
  separate `registry` cache ref — a distinction that silently costs you cache hits if you
  assume `inline` covers everything[^3].
- **Rootless works but is finicky.** Running without root needs user-namespace setup
  (`newuidmap`/`newgidmap`), and some features degrade; see `docs/rootless.md`.
- **The daemon is a gRPC service, not multi-tenant by design.** Exposing it over TCP for a
  shared build farm needs mTLS and careful load balancing (consistent hashing on cache keys,
  or you lose cache locality across the fleet).
- **`gha` / `s3` / `azblob` cache backends were long marked experimental** — behavior and flags
  have shifted between releases, so pin BuildKit versions in CI rather than tracking `latest`.
- **containerd worker vs OCI worker** changes where images land (containerd namespace vs
  BuildKit's internal store) and interacts with whether `docker`/`ctr` can see the result —
  a common surprise when wiring BuildKit into a containerd-based cluster.

## When to Use / When Not

**Use when:**
- You want the modern `docker build` cache model (parallel stages, `RUN --mount=type=cache`,
  cross-runner cache export) — you already have it; learn to configure it.
- You are building images in CI/Kubernetes and want daemon-managed caching without the full
  Docker Engine.
- You are building a higher-level build/CI tool and want a proven engine + LLB as your IR
  rather than shelling out to `docker`.

**Avoid / look elsewhere when:**
- You need daemonless, unprivileged image builds inside a restricted Kubernetes pod — Kaniko
  or Buildah fit that constraint more directly.
- You just want to write a Dockerfile — you don't need this repo at all; use `docker build`.
- You want a high-level, language-native pipeline DSL — use something built *on* BuildKit
  (Dagger, Earthly) instead of driving `buildctl` by hand.

## Alternatives

- `GoogleContainerTools/kaniko` — daemonless, unprivileged Dockerfile builds that run as a job
  inside the cluster; use when you cannot run a privileged build daemon.
- `containers/buildah` — daemonless image construction in the Podman/RHEL ecosystem, scriptable
  beyond Dockerfiles; use on Podman hosts or when you want per-step shell control.
- `earthly/earthly` — a Makefile-like build DSL built on top of BuildKit; use when you want
  repeatable local+CI builds without hand-writing LLB or Dockerfiles.
- `dagger/dagger` — programmable CI/CD engine (also BuildKit-based) with real-language SDKs;
  use when pipelines-as-code matters more than image building alone.
- `moby/moby` — the legacy builder BuildKit replaced; relevant only for understanding history
  or very old engines without BuildKit.

## History

| Version | Date | Notes |
|---------|------|-------|
| proposal | 2017-05 | Split out of moby/moby to replace the legacy image builder[^1]. |
| default in Engine | 2019 | `DOCKER_BUILDKIT=1` opt-in in Docker 18.09; matured over following releases. |
| default `docker build` | 2023-02 | Docker Engine 23.0 makes Buildx + BuildKit the default builder[^2]. |
| v0.11 | 2023-01 | Build attestations: SLSA provenance, SBOM, OCI 1.1 referrers[^4]. |
| v0.13 | 2024 | Windows containers (containerd worker) support introduced as experimental; history/GRPC APIs expanded. |

## References

[^1]: BuildKit proposal in moby/moby — https://github.com/moby/moby/issues/32925 (2017).
[^2]: Docker Engine 23.0 release — Buildx/BuildKit as the default builder. https://docs.docker.com/build/architecture/
[^3]: BuildKit README (LLB, frontends, cache modes, standalone-vs-integrated notes). https://github.com/moby/buildkit/blob/master/README.md
[^4]: BuildKit v0.11 attestations (SLSA provenance / SBOM). https://www.docker.com/blog/generating-sboms-for-your-image-with-buildkit/

[#163]: https://github.com/moby/buildkit/issues/163

## Tags

go, container-builder, docker, oci, dockerfile, build-cache, llb, cloud-native, ci-cd, image-build, supply-chain
