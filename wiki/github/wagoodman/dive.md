# wagoodman/dive

> A terminal UI for exploring a Docker/OCI image layer by layer and finding the wasted space inside it.

[GitHub repo](https://github.com/wagoodman/dive) ·
[License: MIT](https://github.com/wagoodman/dive/blob/main/LICENSE)

## Overview

dive is a single-binary Go tool that opens a Docker or OCI image and lets you walk its layers interactively, showing the file tree each layer adds, modifies, or deletes and estimating how many bytes the image wastes[^1]. It fills a narrow but well-defined niche: `docker history` tells you how big each layer is, but dive tells you *which files* inside each layer are eating the space and where the same bytes were written twice. For most engineers it is the default reach-for tool when a container image is inexplicably large.

The project is largely the work of one primary maintainer (Alex Goodman) and has been self-described as "beta quality" since its early days[^2]. Despite ~54k stars and near-ubiquitous use in CI pipelines and Dockerfile tuning, it has never cut a 1.0 release and remains on the 0.x line. Development is real but low-velocity — the default branch still sees commits (last push 2025-12-15), yet more than 200 issues sit open and releases are infrequent. Treat it as a mature, stable-in-practice utility with a thin bus factor rather than an actively expanding product.

The defining tension: dive is a *read-only diagnostic* that depends on privileged access to a container runtime. To inspect an image it talks to the Docker daemon socket (or Podman), which on most setups is root-equivalent access. That coupling is the source of nearly every operational wrinkle below.

## Getting Started

```bash
# Homebrew (macOS)
brew install dive
# also: apt/deb, pacman -S dive, Nix, Chocolatey/scoop/winget on Windows,
# or: go install github.com/wagoodman/dive@latest
```

Analyze an existing image by tag, id, or digest:

```bash
dive nginx:latest
```

Build and immediately analyze in one step (drop-in for `docker build`):

```bash
dive build -t my-app:dev .
```

Run non-interactively in CI — the UI is skipped and dive exits non-zero if the image fails your thresholds:

```bash
CI=true dive my-app:dev
```

with a `.dive-ci` at the repo root:

```yaml
rules:
  lowestEfficiency: 0.95         # fail if efficiency drops below 95%
  highestWastedBytes: 20MB       # fail if wasted space >= 20MB
  highestUserWastedPercent: 0.20 # fail if wasted space >= 20% of image
```

## Architecture / How It Works

dive does not need a running container. It asks the container engine to export the image as a tar stream (the `docker save` / OCI layout format), then reads each layer tarball itself. For every layer it reconstructs the file tree and diffs it against the accumulated tree of all lower layers, classifying each path as added, modified, removed, or unmodified. Deletions are detected from the union-filesystem whiteout convention (`.wh.` marker files) that Docker writes when a file present in a lower layer is removed in a higher one.

The "image efficiency" score is the tool's one opinionated heuristic. Because a layered image stores every layer's bytes even when a higher layer overwrites or deletes a file, the same logical file can be paid for multiple times. dive sums the bytes of files that are duplicated or superseded across layers and reports both a percentage score and a total wasted-space figure. It is explicitly labeled experimental[^1]: it counts physical duplication, not semantic intent, so it cannot distinguish a deliberately cached artifact from an accidental leftover.

Image sources are pluggable via `--source` (or a `source://image` prefix): `docker` (via the daemon socket, the default), `docker-archive` (a saved tar on disk, requiring no daemon), and `podman` (Linux only). The `docker-archive` source is the escape hatch for environments where you cannot or should not grant socket access.

The interface is a two-pane terminal UI written in Go — layers on the left, combined file tree on the right, with vim-style navigation and toggles to filter the tree by change type. There is no daemon, plugin system, or network service; it is a one-shot CLI that loads, renders, and exits.

## Production Notes

- **Whole-image memory footprint.** dive builds the file trees for every layer in memory to compute the cross-layer diff. On multi-gigabyte images (ML/CUDA bases, monolithic app images) this can consume large amounts of RAM and has been reported to OOM in constrained CI runners. Budget memory proportional to image size, not a fixed ceiling.
- **`dive build` and BuildKit.** The bundled build wrapper analyzes layers produced by the legacy Docker builder. With BuildKit/buildx enabled (the default on modern Docker), the layer output differs and `dive build` may fail or produce nothing useful; the common workaround is to set `DOCKER_BUILDKIT=0` for the build step, or build separately and point dive at the resulting tag.
- **Daemon/socket coupling.** Mounting `/var/run/docker.sock` into the dive container (the documented Docker-based usage) grants that container root-equivalent control of the host daemon. Fine on a laptop; scrutinize it before adding to shared CI. On macOS the daemon runs in a VM, so `dive build` is only supported through the container invocation, not the native binary.
- **`DOCKER_API_VERSION` mismatches.** Against older or non-standard runtimes (Colima, remote contexts, some corporate Docker builds) dive can fail to negotiate the API version; setting `DOCKER_API_VERSION` and/or `DOCKER_HOST` explicitly is the standard fix.
- **Snap install conflicts with apt Docker.** The upstream docs carry an explicit caution that the Snap package can break a Docker daemon installed via `apt-get`[^3]. Prefer the deb/brew/binary installs.
- **CI metrics are coarse.** Only three thresholds exist (efficiency, wasted bytes, wasted percent). There is no per-file allowlist and no awareness of multi-stage builds, so a legitimately large-but-necessary layer can trip the gate; teams typically set loose thresholds and treat dive-CI as a canary, not a hard policy engine.
- **Maintenance cadence.** A slow release cadence and a large open-issue backlog mean edge-case runtime support and feature requests can sit unaddressed for long periods. It is stable because it is simple, not because it is heavily staffed.

## When to Use / When Not

**Use when:**
- You need to find *why* an image is large, down to the file and the layer.
- You want a fast interactive audit while iterating on a Dockerfile.
- You want a lightweight pass/fail size gate in CI without adding a service.
- You're teaching layer/caching mechanics and want a visual of whiteouts and duplication.

**Avoid when:**
- You want automated image *reduction*, not just analysis — dive only reports.
- You need vulnerability or SBOM data — that is a different tool class entirely.
- You're batch-comparing two images programmatically — a non-interactive diff tool fits better than dive's TUI-first design.
- Your runtime can't safely expose a Docker/Podman socket and you can't produce a saved tar for the `docker-archive` source.

## Alternatives

- moby/moby (`docker history` / `docker inspect`) — built into the CLI, gives per-layer sizes and commands but no file-level breakdown; use when you only need a quick "which layer is big" answer.
- GoogleContainerTools/container-diff — non-interactive diff of two images or package/file lists, output-friendly for CI; use when comparing images programmatically rather than exploring one interactively.
- slimtoolkit/slim — analyzes *and* automatically minifies images by profiling what a container actually uses; use when you want reduction, not just a report.
- aquasecurity/trivy — scans image contents for vulnerabilities and generates SBOMs; use when the concern is security/compliance rather than size.
- anchore/syft — SBOM generation and package inventory; use when you need to enumerate what's installed, not where the bytes went.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-05 | First public release; Docker layer explorer TUI[^4]. |
| 0.x | 2018–2026 | Long-lived 0.x line: podman + docker-archive sources, CI mode, `.dive-ci` thresholds, configurable keybindings. Never promoted to 1.0; still self-labeled beta[^2]. |

*(dive tags releases but has remained pre-1.0 throughout; consult the releases page for the current version rather than relying on a fixed number here.)*

## References

[^1]: dive README — features and experimental image-efficiency metric. https://github.com/wagoodman/dive#basic-features
[^2]: dive README — "This is beta quality!" self-description. https://github.com/wagoodman/dive
[^3]: dive README installation caution re: Snap vs apt-installed Docker; upstream issue #546. https://github.com/wagoodman/dive/issues/546
[^4]: Repository metadata, created 2018-05-13. https://github.com/wagoodman/dive

## Tags

go, docker, oci, cli, tui, container-image, image-optimization, devops, ci, docker-layers
