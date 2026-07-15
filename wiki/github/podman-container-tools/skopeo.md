# podman-container-tools/skopeo

> Daemonless, rootless CLI for copying, inspecting, signing, and deleting container images directly against registries.

[GitHub repo](https://github.com/podman-container-tools/skopeo) ·
[License: Apache-2.0](https://github.com/podman-container-tools/skopeo/blob/main/LICENSE)

## Overview

skopeo is a command-line tool for working with container images and registries without a daemon and without root. It copies images between registries and storage backends, inspects remote images without pulling them to disk, deletes images, syncs repositories for air-gapped mirrors, and signs/verifies image content. It began in 2016 at Red Hat's Project Atomic — originally to answer "what's in this registry image?" without a `docker pull`[^1] — and grew into the general-purpose registry Swiss-army knife of the Podman ecosystem.

Its defining characteristic is what it *doesn't* do: it has no image builder, no container runtime, and no background service. skopeo is a thin CLI over the `containers/image` Go library and `containers/storage`, the same libraries that back Podman, Buildah, and CRI-O[^2]. That coupling is the whole story — skopeo's registry auth, TLS, mirror config, and signature policy behave identically to Podman because it *is* Podman's plumbing, exposed as a standalone binary. The upside is consistency; the downside is that skopeo inherits the `/etc/containers` + `$XDG_RUNTIME_DIR` configuration model, which surprises users arriving from `docker`.

Note on naming: the canonical GitHub path is now `podman-container-tools/skopeo` (GitHub redirects the long-standing `containers/skopeo` URL here). Nearly all documentation, image references (`quay.io/skopeo/stable`), and ecosystem discussion still call it `containers/skopeo`; both refer to the same project.

## Getting Started

```bash
# Fedora/RHEL:      sudo dnf install skopeo
# Debian/Ubuntu:    sudo apt install skopeo
# macOS:            brew install skopeo
# Or run the image: quay.io/skopeo/stable
```

```bash
# Inspect a remote image without pulling it
skopeo inspect docker://registry.fedoraproject.org/fedora:latest

# Copy between registries (no local pull, no privilege) — --all keeps every arch
skopeo copy --all docker://quay.io/buildah/stable \
  docker://registry.internal.example.com/buildah

# Convert a registry image into an OCI layout on disk
skopeo copy docker://alpine:latest oci:alpine-oci:latest
```

## Architecture / How It Works

skopeo is deliberately small; the work lives in `containers/image`. The central abstraction is the **transport**: a scheme prefix that tells the library how to read or write an image. `docker://` is a Registry HTTP API V2 endpoint; `dir:` is an unpacked directory of manifest + layer tarballs; `oci:` is an OCI image-layout directory; `containers-storage:` is the local Podman/CRI-O/Buildah store; `docker-archive:` is a `docker save` tarball; `docker-daemon:` talks to a running Docker daemon. Every command is essentially "read from transport A, write to transport B," which is why the same tool copies, converts formats, and loads/saves in one code path.

Because it reuses `containers/image`, skopeo's runtime behavior is governed by the shared `/etc/containers` config files rather than anything skopeo-specific: `registries.conf` (mirrors, short-name aliases, insecure registries), `policy.json` (signature verification policy — by default `insecureAcceptAnything`), and `registries.d` (where signatures are stored/fetched). Credentials come from `$XDG_RUNTIME_DIR/containers/auth.json` written by `skopeo login` (or `podman`/`buildah`/`docker login`), with `REGISTRY_AUTH_FILE` and `~/.docker/config.json` as fallbacks.

Signing spans two eras. The original "simple signing" uses GPG detached signatures stored out-of-band via `registries.d`. Newer sigstore support (`skopeo generate-sigstore-key`, `--sign-by-sigstore`) produces cosign-compatible signatures. Verification is enforced through `policy.json`, not command flags — a point of confusion for people expecting `--verify` to gate a copy.

## Production Notes

- **Multi-arch is opt-in.** `skopeo copy` without `--all` copies only the manifest matching the host platform. Mirroring a multi-arch image (or an image index / manifest list) without `--all` silently produces a single-arch mirror — a frequent air-gap footgun. Use `--all` for faithful mirrors.
- **`delete` doesn't reclaim space.** `skopeo delete` marks the manifest for removal; the registry must have deletion enabled and run its own garbage collection before storage is freed. Against registries that disable delete, it simply errors.
- **Static/portable builds need build tags.** Building skopeo from source pulls in cgo dependencies (`gpgme`, storage graph drivers). Portable or non-Linux builds require tags like `containers_image_openpgp` (pure-Go OpenPGP, drops the gpgme cgo dep) and graphdriver exclusions. This trips up anyone vendoring or cross-compiling it[^3].
- **Config lives in `/etc/containers`, not `~/.docker`.** Mirror routing, insecure-registry allowances, and short-name resolution are read from `registries.conf`. A `docker pull` that works may fail under skopeo (or vice versa) purely because they consult different config.
- **Flaky-registry handling.** `--retry-times` (and per-request retry logic in `containers/image`) matters in CI against rate-limited registries like Docker Hub; without it, transient 429/5xx aborts the copy.
- **`sync` for air-gapped flows.** `skopeo sync --src/--dest` with a YAML source list mirrors whole repositories to `dir:` or another registry — the intended tool for USB-drive / disconnected transfers, but the YAML schema and tag-filtering semantics are easy to get subtly wrong.

## When to Use / When Not

**Use when:**
- Mirroring or promoting images between registries in CI without a Docker daemon or root.
- Inspecting tags, labels, digests, and layers of a remote image before pulling it.
- Converting between image formats (registry ↔ OCI layout ↔ docker-archive).
- Air-gapped / disconnected image transfer, or scripted signing and verification.

**Avoid when:**
- You need to *build* images — use Buildah or a Dockerfile builder.
- You need to *run* containers — use Podman or Docker.
- You want image operations *inside* a Go program — call `containers/image` directly instead of shelling out.
- You want to stand up a registry — that's the server side (Distribution), not skopeo.

## Alternatives

- google/go-containerregistry — `crane`/`gcrane` CLI plus a Go library; copy/inspect/mirror with no `/etc/containers` config model. Use when you want a self-contained tool outside the Podman ecosystem.
- regclient/regclient — `regctl` daemonless client with strong manifest and OCI-referrers manipulation. Use when you need fine-grained manifest surgery.
- oras-project/oras — push/pull for arbitrary OCI *artifacts*, not just runnable images. Use when storing non-image content (charts, SBOMs, WASM) in registries.
- containers/image — the library skopeo is built on. Use when you need these operations embedded in Go, not as a subprocess.
- distribution/distribution — the registry *server*. Use when you need to host images, not move them.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2016-03 | Created at Red Hat / Project Atomic to inspect registry images without pulling[^1]. |
| 1.0.0 | 2020-03 | First stable release; CLI and transports considered stable[^4]. |
| sigstore support | 2022 | `generate-sigstore-key` and sigstore/cosign-style signing added. |
| Current | 2026-07 | Actively maintained; ~11.1k stars, part of the Podman/containers toolchain. |

## References

[^1]: skopeo README and project history — https://github.com/containers/skopeo
[^2]: `containers/image`, the shared image-transport library used by Podman, Buildah, CRI-O, and skopeo — https://github.com/containers/image
[^3]: skopeo install/build documentation, including build tags for portable binaries — https://github.com/containers/skopeo/blob/main/install.md
[^4]: skopeo releases — https://github.com/containers/skopeo/releases

## Tags

go, containers, oci, docker-registry, container-images, devops, cli, image-signing, supply-chain, podman-ecosystem, air-gapped
