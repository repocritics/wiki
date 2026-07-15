# earthly/earthly

> A container-native build framework with a Dockerfile-meets-Makefile syntax — now frozen after the company wound down its commercial arm in 2025.

[GitHub repo](https://github.com/earthly/earthly) ·
[Official website](https://earthly.dev) ·
[License: MPL-2.0](https://github.com/earthly/earthly/blob/main/LICENSE)

## Overview

Earthly is a build automation tool that runs every build step inside a container, using a declarative file (the `Earthfile`) whose syntax deliberately blends Dockerfile commands (`FROM`, `RUN`, `COPY`) with Makefile-style named targets. It was created by Earthly Technologies in 2020[^1] and positioned as a glue layer between language-specific build tooling (maven, gradle, npm, `go build`) and CI systems, giving reproducible, cache-aware, parallelized builds that behave identically on a laptop and in CI.

The defining fact about Earthly in 2026 is not its design but its status. The README states plainly that the project "is no longer actively maintained," pointing to the company's decision to shut down its Earthfiles/cloud commercial products[^2]. The last substantive push to `main` was in late 2025. The stars (~12k) reflect a tool that earned genuine adoption during 2021–2023; the open-issue count (700+) and the maintenance freeze reflect a codebase that is stable but no longer receiving fixes or features. Anyone evaluating Earthly today is evaluating a frozen artifact, not a living project.

The underlying tension was always Earthly's business model. The open-source CLI was free and self-hostable, but the fast-path experience — shared remote caching via Earthly Satellites — was a paid cloud service. When the cloud business did not become sustainable, the OSS tool lost its funded maintainers along with it.

## Getting Started

```bash
# macOS/Linux install script (pins a specific release)
sudo /bin/sh -c 'wget https://github.com/earthly/earthly/releases/latest/download/earthly-linux-amd64 -O /usr/local/bin/earthly && chmod +x /usr/local/bin/earthly && /usr/local/bin/earthly bootstrap'
# or: brew install earthly && earthly bootstrap
```

Earthly requires a running Docker (or Podman) daemon; it launches its own BuildKit instance in a container on first use.

```earthly
# Earthfile
VERSION 0.8
FROM golang:1.21-alpine
WORKDIR /app

build:
  COPY main.go .
  RUN go build -o build/app main.go
  SAVE ARTIFACT build/app AS LOCAL build/app   # write artifact back to host

docker:
  COPY +build/app .                            # pull artifact from another target
  ENTRYPOINT ["/app/app"]
  SAVE IMAGE app:latest
```

Invoke with `earthly +docker`. Targets are referenced with the `+` sigil; `+build/app` means "the `app` artifact produced by the `build` target."

## Architecture / How It Works

Earthly is a frontend over [moby/buildkit](https://github.com/moby/buildkit) — the same engine behind modern `docker build`[^3]. An Earthfile is parsed into a set of targets; Earthly compiles those targets into a BuildKit LLB (low-level build) graph, then hands it to BuildKit for execution. This is why Earthly's caching semantics feel identical to Docker layer caching: they are Docker layer caching.

The two ideas Earthly adds on top of BuildKit are:

1. **Targets and cross-target references.** Each target is an isolated build unit. `COPY +other-target/artifact ./` and `FROM +base-target` let one target consume another's outputs by content hash, forming a directed acyclic graph that Earthly executes with independent branches in parallel. `SAVE ARTIFACT ... AS LOCAL` and `SAVE IMAGE` are the two escape hatches back to the host — the features that make Earthly a general build tool rather than just an image builder.
2. **A cross-repository import system.** `earthly github.com/org/repo+target` resolves and builds a target from another repository without cloning it manually. This is Earthly's answer to monorepo/polyrepo build reuse and has no direct Dockerfile equivalent.

The `VERSION` pragma at the top of every Earthfile pins the language/feature set (0.5 through 0.8 over the project's life), and Earthly gates behavior changes behind it to preserve older Earthfiles. Remote caching — the performance story that made large teams adopt Earthly — could target a registry (`--remote-cache`) or, on the paid path, Earthly Satellites: persistent remote BuildKit workers that kept a warm cache between CI runs.

## Production Notes

- **Maintenance is the headline caveat.** New Docker/BuildKit releases, new base-image formats, or a CVE in a bundled dependency will not be met with an upstream fix. Treat the current release as the last one and pin it.
- **Satellites are effectively gone.** Earthfiles that relied on Satellite-backed shared caching lose their main speed advantage against a cold local cache. Self-hosted remote cache via a registry still works but is slower and requires manual setup.
- **A daemon is mandatory.** Earthly cannot run without a container runtime and runs a privileged BuildKit container. In restricted CI (no Docker-in-Docker, no privileged mode) it is awkward or impossible to run. This was a recurring friction point even during active development.
- **Cache is powerful but opaque.** Non-obvious cache invalidation — a changed file high in the target graph rebuilding everything below it — is the most common source of "why is this slow" confusion. `--no-cache` and interactive debugging (`--interactive`) exist but the mental model takes time.
- **Not a lock-in-free format.** Earthfiles are not Dockerfiles. `FROM DOCKERFILE .` can consume an existing Dockerfile, but there is no clean automated path to convert an Earthfile *back* to portable tooling if you later abandon Earthly. Given the frozen status, this migration risk is now the central adoption question.

## When to Use / When Not

**Use when:**
- You already have Earthly in production and it works — the tool is stable and the freeze does not break existing pipelines.
- You want reproducible container-based builds with parallelism and want to learn one syntax that reads like Dockerfile + Make.
- You are studying BuildKit frontends or want a self-contained example of an LLB-generating build tool.

**Avoid when:**
- You are starting a new project in 2026 — adopting an unmaintained build tool as core infrastructure is hard to justify when Dagger, Nix, or plain BuildKit/Bake are maintained.
- Your CI cannot run a privileged container / Docker-in-Docker.
- You need long-term vendor support, security patching, or a roadmap.

## Alternatives

- dagger/dagger — the closest spiritual successor; container-native CI/CD as code (Go/Python/TS SDKs) from an overlapping design lineage. Use instead when you want a maintained tool with a similar "builds run in containers" model.
- moby/buildkit — Earthly's own engine, usable directly via `buildx bake` and HCL/JSON build definitions. Use when you want the caching foundation without Earthly's target layer.
- NixOS/nix — content-addressed, fully reproducible builds without containers. Use when reproducibility and hermeticity matter more than a gentle learning curve.
- bazelbuild/bazel — a full build system with fine-grained dependency tracking for large monorepos. Use when you need correctness at scale and can absorb the setup cost.
- casey/just — a simple command runner (a saner Make). Use when you only wanted convenient named targets, not containerized reproducibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-03 | Public release of Earthly and the Earthfile format[^1]. |
| Satellites era | 2021–2023 | Cloud remote build/cache product; peak adoption. `VERSION` 0.6/0.7 feature sets. |
| 0.8 | ~2024 | Current Earthfile feature set referenced in the README. |
| wind-down | 2025 | Company shuts down Earthfiles/cloud products; repo enters maintenance freeze[^2]. Last push 2025-10. |

## References

[^1]: earthly/earthly repository, created 2020-03-12. https://github.com/earthly/earthly
[^2]: "Shutting Down Earthfiles Cloud" — Earthly blog, linked from the repository README's maintenance notice. https://earthly.dev/blog/shutting-down-earthfiles-cloud
[^3]: moby/buildkit — the LLB build engine Earthly compiles to. https://github.com/moby/buildkit

## Tags

go, build-system, ci-cd, containers, buildkit, docker, reproducible-builds, build-automation, monorepo, unmaintained, developer-tools
