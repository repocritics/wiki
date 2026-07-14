# podman-container-tools/buildah

> Daemonless CLI and Go library for building OCI/Docker container images — with or without a Dockerfile, as a normal user.

[GitHub repo](https://github.com/containers/buildah) ·
[Official website](https://buildah.io) ·
[License: Apache-2.0](https://github.com/containers/buildah/blob/main/LICENSE)

## Overview

Buildah is a command-line tool and Go library for assembling
[Open Container Initiative](https://opencontainers.org/) images. It sits in the
"containers" family of tools (Podman, Skopeo, CRI-O) that Red Hat began as
Project Atomic and that now share a common set of libraries. Its defining
premise is that image building does not need a long-running privileged daemon:
each `buildah` invocation is an ordinary fork-exec process, and builds can run
rootless inside a user namespace[^1]. This is the direct rebuttal to the classic
Docker model where a root `dockerd` mediates every build.

Buildah offers two build styles from one binary. The first is Dockerfile-compatible:
`buildah build` (originally `buildah bud`, "build-using-dockerfile") consumes a
Containerfile or Dockerfile and produces an image the same way `docker build`
would. The second is a scriptable, coreutils-style API — `buildah from`,
`buildah run`, `buildah copy`, `buildah config`, `buildah commit` — where each
image layer is an explicit shell step. That second mode is the project's real
differentiator: you can build an image with no Dockerfile at all, mount a working
container's root filesystem, edit it with any tool on the host, and commit the
result[^1].

The central tension is scope. Buildah deliberately does not run, manage, or serve
containers — that is Podman's job, and `podman build` is in fact a thin wrapper
over Buildah's `imagebuildah` package[^2]. Buildah is the build primitive; you
reach for it directly when you want fine-grained, script-driven image assembly,
and you reach for Podman when you want one CLI that also runs and manages what you
built. Note the GitHub path now resolves to `podman-container-tools/buildah`; the
canonical import path and docs remain `github.com/containers/buildah`[^3].

## Getting Started

```bash
# Fedora / RHEL / CentOS Stream
sudo dnf -y install buildah
# Debian / Ubuntu
sudo apt-get -y install buildah
```

```bash
# 1. Dockerfile-compatible path
buildah build -t myimage:latest .

# 2. Dockerfile-free, scriptable path
ctr=$(buildah from alpine:latest)
buildah run "$ctr" -- apk add --no-cache curl
buildah config --entrypoint '["/usr/bin/curl"]' "$ctr"
buildah commit "$ctr" myimage:latest

# push to a registry (auth shared with podman/skopeo)
buildah push myimage:latest docker://registry.example.com/me/myimage:latest
```

## Architecture / How It Works

Buildah is mostly glue over three shared Go libraries that the whole containers
stack vendors:

- **containers/storage** — the local image and container store (overlay, fuse-overlayfs,
  or vfs graph drivers). Determines how layers are stacked and where they live on disk.
- **containers/image** — pulling, pushing, signing, and format conversion between OCI
  and Docker v2s2 manifests; also the registry-auth handling.
- **containers/common** — shared configuration (`containers.conf`, `storage.conf`,
  `policy.json`, registries).

Because these are the same libraries Podman and Skopeo use, a Buildah-built image,
a Podman-run container, and a Skopeo-copied image all agree on storage layout and
registry auth. That shared foundation is the practical reason the tools compose so
cleanly.

For `buildah run` and for `RUN` steps in a Containerfile, Buildah shells out to an
OCI runtime — `crun` (default on most modern installs) or `runc`. Rootless mode
relies on `/etc/subuid` and `/etc/subgid` range maps plus user namespaces;
`buildah unshare` drops you into that namespace so you can manipulate storage as if
root. There is no daemon and no persistent server state — everything is a process
that acquires a lock on the store, does its work, and exits[^1].

Layer caching is opt-in rather than default. `--layers` (or `BUILDAH_LAYERS=true`)
enables Dockerfile-style intermediate-layer caching; without it, a `buildah build`
commits a single squashed layer per stage. This is a deliberate divergence from
Docker's always-on cache and a frequent source of "why is my cache not hitting"
confusion for people migrating.

## Production Notes

**Rootless storage drivers.** Rootless builds historically required `fuse-overlayfs`
because unprivileged overlay mounts were unsupported; recent kernels (5.11+) plus a
current containers stack allow native rootless overlay, but on older hosts you fall
back to `fuse-overlayfs` or, worst case, the `vfs` driver. `vfs` does a full copy of
the filesystem per layer — it works anywhere but is slow and disk-hungry, and it is
a classic footgun in constrained CI containers where overlay is unavailable[^4].

**Building inside a container (CI).** Running Buildah in a container (buildah-in-a-container)
needs a carefully relaxed sandbox: appropriate `--security-opt`, seccomp/apparmor
handling, and often extra user-namespace configuration. The maintained
`quay.io/buildah/stable` image documents the working invocation; ad-hoc setups tend
to fail on `mount` or user-namespace errors[^5].

**Storage location and root/rootless split.** Rootless data lives under
`~/.local/share/containers/storage`; root data under `/var/lib/containers/storage`.
Images built as root are not visible to a rootless invocation and vice versa — a
common "my image disappeared" surprise when a script mixes `sudo` and non-`sudo`
calls.

**Cross-architecture builds.** `--platform` / `--arch` builds for a foreign
architecture require `qemu-user-static` binfmt handlers registered on the host;
without them, `RUN` steps for the foreign arch fail at exec time.

**Not full BuildKit.** Buildah has grown build-time bind/cache/secret/ssh mounts and
heredoc support, but it is not a BuildKit reimplementation — there is no LLB frontend,
no distributed build graph, and caching semantics differ. Teams expecting `docker
buildx` feature parity should validate specific flags rather than assume equivalence.

## When to Use / When Not

**Use when:**
- You want rootless, daemonless image builds in CI or on locked-down hosts.
- You need to build images without a Dockerfile, or to script layer construction
  with arbitrary host tooling.
- You are already in the Podman/Skopeo ecosystem and want the underlying build primitive.
- You want a Go library to embed image-building into your own tool.

**Avoid when:**
- You need BuildKit's advanced caching, frontends, or distributed builds.
- You want a single tool that also runs and orchestrates containers day-to-day —
  use Podman (which uses Buildah underneath) or Docker.
- You are on Windows/macOS without a Linux VM — Buildah is Linux-native and runs
  elsewhere only through a Linux virtual machine.

## Alternatives

- moby/buildkit — use when you want the most advanced caching, build secrets, and
  frontends, and are willing to run a build backend/daemon.
- containers/podman — use when you want one CLI for build + run + pods; `podman build`
  wraps Buildah, so the build behavior is the same.
- GoogleContainerTools/kaniko — use when building images inside a Kubernetes cluster
  without a privileged daemon.
- containers/skopeo — complementary, not a builder: use to inspect, copy, and sign
  images between registries and local storage.
- ko-build/ko — use when building minimal images for Go apps declaratively, skipping
  Dockerfiles entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-01-26 | Repository created under the containers (ex-Project Atomic) org[^3]. |
| 1.0 | 2018 | First stable release; establishes the `buildah bud` + coreutils workflow. |
| 1.x | 2019–2026 | Rolling minor releases; `buildah build` added as alias for `bud`; rootless, `--layers` caching, build-time mounts, heredoc support land incrementally. |
| — | 2026-07 | GitHub path resolves to `podman-container-tools/buildah`; import path stays `github.com/containers/buildah`[^3]. |

## References

[^1]: Buildah README — build model, daemonless/rootless design, Buildah–Podman relationship. https://github.com/containers/buildah/blob/main/README.md
[^2]: Podman documentation — `podman build` uses Buildah's Go API for image builds. https://docs.podman.io/en/latest/markdown/podman-build.1.html
[^3]: `gh api repos/containers/buildah` (resolved 2026-07-15): full_name `podman-container-tools/buildah`, created 2017-01-26, Apache-2.0, Go, ~8.9k stars, homepage buildah.io.
[^4]: Buildah troubleshooting guide — storage drivers, rootless overlay vs fuse-overlayfs vs vfs. https://github.com/containers/buildah/blob/main/troubleshooting.md
[^5]: Buildah in a container image and usage notes. https://github.com/containers/image_build/blob/main/buildah/README.md

## Tags

go, containers, oci, container-image, image-build, dockerfile, rootless, daemonless, podman, devops, cli, linux
