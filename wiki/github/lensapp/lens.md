# lensapp/lens

> The Lens Desktop Kubernetes IDE — an Electron client for browsing and operating clusters through your local kubeconfig. The open-source build in this repo has been retired.

[GitHub repo](https://github.com/lensapp/lens) ·
[Official website](https://k8slens.dev/) ·
[License: MIT](https://github.com/lensapp/lens/blob/master/LICENSE)

## Overview

Lens is a cross-platform desktop application (Electron + TypeScript) that presents a graphical dashboard over one or more Kubernetes clusters. It reads the kubeconfig contexts on your machine, connects to each cluster's API server with your existing credentials, and renders workloads, config, networking, storage, RBAC, custom resources, events, logs, and metrics in a single window with a built-in terminal. It originated as "Kontena Lens" and was open-sourced around 2019; Kontena and Lens were acquired by Mirantis in 2020[^1], after which the product grew a commercial track.

The defining fact for anyone evaluating this repo today is that **the open-source version is no longer maintained**. The repository's README states the open-source Lens Desktop has been retired, that development continues only in the closed commercial product, and that outside contributions are now accepted through the Lens extension API rather than to the core[^2]. The default branch (`lens-desktop`) carries this notice; the legacy source lives on the `master` branch, last meaningfully pushed in early 2025. The ~23k stars reflect Lens's years as the dominant Kubernetes desktop UI, not current activity.

The other defining tension is commercialization. With Lens 6 (2022) the application began requiring a Lens ID sign-in to launch, and several capabilities moved behind proprietary extensions and paid tiers (Lens Pro, Lens Enterprise)[^3]. This drove a persistent community split: unofficial "OpenLens" binaries built from the open parts, and, after those wound down, an active hard fork, Freelens.

## Getting Started

The maintained product is a signed installer, not a build target:

```bash
# macOS via Homebrew cask (Mirantis Lens Desktop)
brew install --cask lens
```

On first launch Lens auto-detects `~/.kube/config`, lists each context in its catalog, and connects on click. There is no config file to write for the basic case — it inherits whatever `kubectl` already uses:

```bash
# Lens simply reuses the same contexts kubectl sees
kubectl config get-contexts
kubectl config use-context my-cluster   # now visible in Lens's catalog
```

Building the retired open-source tree from `master` is possible but unsupported and produces an unbranded build missing the proprietary extensions. Most users who want an OSS path now install Freelens instead (see Alternatives).

## Architecture / How It Works

Lens is a client, not a server. It holds no cluster credentials of its own — it acts as your kubeconfig, using the same client certificates, tokens, or exec-plugin (OIDC, `aws eks get-token`, `gke-gcloud-auth-plugin`) auth that `kubectl` would. Everything it shows is a read or write against the cluster's API server under your identity, so its RBAC is exactly your RBAC.

Internals worth knowing:

- **Electron split.** A main process manages cluster connections, kubeconfig watching, and a bundled `kubectl`; renderer processes draw the dashboard. Each connected cluster runs as an isolated frame.
- **Watch-based UI.** Resource lists are backed by API `watch` streams, so the dashboard updates live. On large clusters this means many concurrent watches and correspondingly high memory/CPU in the desktop app.
- **Bundled kubectl + terminal.** Lens ships its own `kubectl` binary (version-matched per cluster where possible) and an integrated terminal pre-pointed at the active context.
- **Metrics.** Node/pod CPU and memory charts require an in-cluster Prometheus. Lens can install its own "Lens Metrics" Prometheus stack, or be pointed at an existing one; without it the graphs are empty.
- **Extension API.** A VS Code-style extension model lets third parties add catalog entries, cluster pages, and status-bar items. Post-retirement this is the only supported way to extend the product, and some formerly built-in features now ship as (in cases proprietary) extensions.

The coupling story: the open UI shell is genuinely a thin client over the Kubernetes API, but the *product* increasingly depends on Mirantis-hosted identity (Lens ID) and closed extensions, so the OSS shell and the shipping app have diverged.

## Production Notes

- **Sign-in requirement is the top footgun.** Since v6 the desktop app requires a Lens ID account to use. This is a hard blocker for airgapped, regulated, or offline environments and is the single most common reason teams migrate off Lens. Verify current sign-in behavior for your version before standardizing on it.
- **The repo is retired — do not build tooling on it.** Treat `lensapp/lens` as archival. New issues against the core are effectively unmaintained; the commercial product is where fixes land, and the community fork is where OSS development continues.
- **It runs as you.** Lens performs deletes, edits, scale operations, and `exec` into pods with your credentials and no additional confirmation gate beyond the UI. On production clusters, an accidental edit or delete in the dashboard is a real change. Prefer least-privilege kubeconfig contexts.
- **Resource weight.** As an Electron app watching every listed resource, Lens is heavy on clusters with thousands of pods/objects; the terminal-based k9s is dramatically lighter for the same browsing tasks.
- **Metrics are not free.** The CPU/memory charts silently show nothing unless a Prometheus stack is present and reachable; teams often mistake this for a Lens bug.
- **Exec-plugin auth.** Clusters using cloud auth plugins require those binaries (`aws`, `gke-gcloud-auth-plugin`, etc.) on PATH; Lens shells out to them exactly as kubectl does, so a missing plugin fails the connection.

## When to Use / When Not

**Use when:**
- You want a graphical, multi-cluster dashboard and are comfortable with a commercial Mirantis product (and its sign-in).
- Your team is learning Kubernetes and benefits from a visual view of workloads, logs, and events.
- You already pay for Lens Pro/Enterprise and want the supported feature set and 24/5 support.

**Avoid when:**
- You need an actively maintained *open-source* desktop UI — this repo is retired; use Freelens or Headlamp.
- You operate airgapped/offline or cannot accept a mandatory account sign-in.
- You want minimal footprint for cluster browsing — a terminal UI (k9s) is faster and lighter.
- You need a shared, server-hosted dashboard for a team rather than a per-user desktop client.

## Alternatives

- derailed/k9s — terminal UI for Kubernetes; use instead when you want a fast, keyboard-driven, low-footprint client with no sign-in.
- freelensapp/freelens — community hard fork continuing the open Lens Desktop; use instead when you specifically want the Lens experience under an OSS, no-account model.
- kubernetes/dashboard — the official web dashboard; use instead when you want a browser-based, in-cluster deployment rather than a desktop app.
- kubernetes-sigs/headlamp — extensible web/desktop UI under CNCF; use instead when you want an actively maintained, vendor-neutral OSS dashboard with an extension model.
- Aptakube (closed source) — lightweight multi-cluster desktop client; use instead when you want a Lens-like desktop UX without Electron weight or mandatory sign-in.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-11-12 | Repository created (as Kontena Lens)[^4]. |
| — | 2020 | Kontena/Lens acquired by Mirantis; product continues as OSS[^1]. |
| 5.0 | 2021 | Catalog / Lens Spaces; multi-cluster management additions. |
| 6.0 | 2022 | Lens ID sign-in required to launch; Lens Pro tier introduced[^3]. |
| — | 2022–2023 | "OpenLens" community builds of the open parts circulate as sign-in-free alternative. |
| — | 2024–2025 | Open-source Lens Desktop retired; contributions moved to the extension API; repo becomes archival[^2]. |

## References

[^1]: Mirantis, "Mirantis acquires Lens and Kontena" (2020). https://www.mirantis.com/company/press-center/company-news/
[^2]: lensapp/lens README, "History of this Repository" — states the open-source Lens Desktop has been retired and contributions move to the extension API. https://github.com/lensapp/lens
[^3]: Lens Desktop pricing and Lens Pro / Lens ID sign-in. https://k8slens.dev/pricing
[^4]: GitHub repository metadata, `lensapp/lens` (created 2018-11-12, MIT, ~23.2k stars, default branch `lens-desktop`, last push 2025-02). https://github.com/lensapp/lens

## Tags

kubernetes, kubernetes-dashboard, desktop-app, electron, typescript, devops, cloud-native, cluster-management, kubernetes-ui, retired
