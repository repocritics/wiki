# fluxcd/flux2

> GitOps continuous delivery for Kubernetes — a set of controllers that reconcile cluster state against Git, OCI, and Helm sources.

[GitHub repo](https://github.com/fluxcd/flux2) ·
[Official website](https://fluxcd.io) ·
[License: Apache-2.0](https://github.com/fluxcd/flux2/blob/main/LICENSE)

## Overview

Flux is a GitOps engine for Kubernetes: it runs inside the cluster and continuously pulls declarative configuration from a source of truth (a Git repository, an OCI artifact, or a Helm repository) and applies it, correcting drift without an external CI pipeline holding cluster credentials[^1]. It is a CNCF graduated project, used across a range of production platforms and cloud vendor integrations[^2].

The current codebase — "Flux v2" — is a ground-up rewrite of the original Weaveworks Flux ("v1", now end-of-life) around Kubernetes' API-extension model[^3]. Where v1 was a single monolithic daemon, v2 is the **GitOps Toolkit**: a set of specialized controllers, each owning a small group of custom resources (CRDs). The `flux2` repo itself is mostly the `flux` CLI and the bootstrap/installation manifests; the actual reconciliation logic lives in sibling repos (`source-controller`, `kustomize-controller`, `helm-controller`, and others) that ship as the Flux distribution.

The defining tension is composability versus turnkey experience. Flux gives you Kubernetes-native primitives — everything is a CRD you can `kubectl get`, reference across namespaces, and wire together with dependency ordering — but it deliberately ships **no dashboard, no application abstraction, and no central control plane**. That is the cleanest expression of "the cluster reconciles itself," and also the reason teams who want a UI-first, application-centric workflow often reach for Argo CD instead.

## Getting Started

Install the CLI (Homebrew, or the install script):

```bash
brew install fluxcd/tap/flux
# or: curl -s https://fluxcd.io/install.sh | sudo bash
```

Check the target cluster is compatible, then bootstrap Flux into a Git repo (it commits its own manifests and reconciles itself thereafter):

```bash
flux check --pre

export GITHUB_TOKEN=<pat>
flux bootstrap github \
  --owner=my-org \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal
```

A minimal source + sync pair (what bootstrap ultimately writes into Git):

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/stefanprodan/podinfo
  ref:
    branch: master
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: default
  sourceRef:
    kind: GitRepository
    name: podinfo
  path: "./kustomize"
  prune: true            # delete resources removed from Git
  wait: true             # block until applied objects are healthy
```

## Architecture / How It Works

Flux has no server and no agent-to-controller RPC. Every component is a Kubernetes controller that watches CRDs and reacts through the API server and events. The toolkit is six controllers:

1. **source-controller** — fetches artifacts from `GitRepository`, `OCIRepository`, `HelmRepository`, `HelmChart`, and `Bucket` resources, verifies them (signatures, checksums), and republishes them as tarballs served over an in-cluster HTTP endpoint. It is the single fetch point; other controllers never talk to Git directly.
2. **kustomize-controller** — reconciles `Kustomization` resources. It builds Kustomize overlays from a source artifact and applies them via **server-side apply**, which is also how it detects and corrects drift. It handles `dependsOn` ordering, health checks (`wait`), pruning, and SOPS decryption of secrets.
3. **helm-controller** — reconciles `HelmRelease` resources by driving the Helm SDK in-process (no Tiller). It manages install/upgrade/rollback, release history, and optional drift correction.
4. **notification-controller** — the events hub. Outbound: `Alert` + `Provider` route reconciliation events to Slack, Teams, Git commit statuses, etc. Inbound: `Receiver` exposes webhooks so a Git push can trigger immediate reconciliation instead of waiting for the poll interval.
5. **image-reflector-controller** — scans container registries and records tags against `ImageRepository` / `ImagePolicy` resources.
6. **image-automation-controller** — writes updated image tags back into Git (`ImageUpdateAutomation`), closing the loop for automated deploys.

Reconciliation is pull-based and interval-driven: each resource declares a `spec.interval`, and controllers requeue on that cadence (webhooks via the notification-controller short-circuit the wait). Because state lives entirely in CRD status and Kubernetes events, Flux inherits the API server's RBAC and audit model rather than inventing its own. Cross-namespace references plus per-`Kustomization` service-account impersonation (`spec.serviceAccountName`) provide **soft** multi-tenancy — isolation is only as strong as your RBAC.

## Production Notes

**There is no official UI.** This is the single most common surprise for teams migrating from Argo CD. Weave GitOps (the OSS dashboard from Weaveworks) was effectively abandoned after Weaveworks shut down in early 2024[^4]. Community options exist — Capacitor is the most cited — but none are a first-party, supported dashboard. Observability in practice is `flux get all -A`, `flux logs`, Prometheus metrics, and Grafana dashboards.

**API version churn hurts on upgrade.** The toolkit CRDs went through `v1beta1` → `v1beta2` → `v1` for the core kinds, and each promotion required manifest updates and sometimes changed defaults. Flux's own controllers manage the CRDs during `flux install` / `flux bootstrap`, so a controller upgrade and a CRD schema change land together; pin the version, read the release notes, and test on a non-prod cluster. Downgrades are not supported.

**Interval tuning is a real cost.** Aggressive `spec.interval` values across many resources multiply Git fetches and API-server writes. On large fleets, source-controller becomes a bottleneck and is commonly sharded (multiple instances by label selector). Prefer webhooks (`Receiver`) over short intervals for fast feedback.

**Bootstrap couples you to a Git provider.** `flux bootstrap github|gitlab|...` needs a token with repo-admin scope and commits directly to your infra repo. The declarative alternative (`flux install` + a manually managed `GitRepository`/`Kustomization` for Flux itself) avoids the credential coupling but you own the sync-self wiring.

**Image automation writes commits.** The image-automation-controller pushes to Git, which can fight noisy CI or create commit loops if the same branch is watched and written. Isolate the automation branch/path and scope the write credentials narrowly.

**Secrets.** There is no built-in secret store; the supported pattern is SOPS-encrypted files in Git decrypted by kustomize-controller (age or a cloud KMS), or an external operator like External Secrets. Losing the decryption key means an unrecoverable sync.

**Helm scale.** helm-controller renders and diffs in-process; very large releases or long histories can drive memory pressure. Bound `maxHistory` and consider drift detection settings per release.

## When to Use / When Not

**Use when:**
- You want GitOps that is Kubernetes-native to the bone — everything a CRD, RBAC and events for free, no extra control plane to secure.
- You need Helm, Kustomize, OCI artifacts, and automated image updates under one reconciler with dependency ordering.
- You want the cluster to pull (no CI system holding cluster-admin credentials).
- You're building a platform and want composable controllers to extend, not a fixed product.

**Avoid when:**
- Your team wants a visual dashboard, application topology view, and click-to-sync out of the box (Argo CD fits better).
- You want an opinionated "Application" abstraction and app-of-apps rollups rather than raw CRDs.
- You manage a few clusters and the CRD/multi-controller surface is more machinery than the problem warrants.
- You need hard multi-tenant isolation from the CD tool itself rather than from cluster RBAC.

## Alternatives

- argoproj/argo-cd — the main competitor; UI-first, application-centric, central control plane. Use it when a dashboard, SSO-backed RBAC, and multi-cluster management from one place matter more than pure Kubernetes-native composability.
- rancher/fleet — GitOps built for very large fleets of clusters, integrated with Rancher. Use it when you're managing hundreds/thousands of downstream clusters from a central hub.
- carvel-dev/kapp-controller — Carvel's app-reconciliation controller (fetch/template/deploy). Use it when you're already in the Carvel toolchain (ytt/kbld/kapp) rather than Kustomize/Helm.
- codefresh (Argo-based) — commercial GitOps platform layered on Argo CD. Use it when you want enterprise support and dashboards without self-hosting the control plane.
- kubernetes-sigs/kustomize + kubectl apply in CI — the non-GitOps baseline. Use it when a push-based pipeline is genuinely simpler than running reconcilers in-cluster.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-04-24 | `flux2` repo created; GitOps Toolkit rewrite begins[^3]. |
| v1 EOL | 2020-11 | Original Weaveworks Flux deprecated in favor of v2; migration required[^3]. |
| graduated | 2022-11-30 | Flux becomes a CNCF graduated project[^2]. |
| 2.0.0 | 2023-07 | First GA release of Flux v2; APIs declared stable[^1]. |
| 2.x | 2023–2026 | Ongoing minor releases: OCI-native sources, sharding, drift-correction and API `v1` promotions. |

## References

[^1]: Flux CD, "Flux is now GA" — July 2023. https://fluxcd.io/blog/2023/07/flux-ga/
[^2]: CNCF, "Flux Continuous Delivery Project Achieves Graduation" — 2022-11-30. https://www.cncf.io/announcements/2022/11/30/flux-cd-graduation/
[^3]: Flux CD, "Migrating from Flux v1 to v2" / project documentation. https://fluxcd.io/flux/migration/
[^4]: The New Stack, "Weaveworks Closes Shop, GitOps Legacy Lives On" — 2024. https://thenewstack.io/weaveworks-closes-shop-gitops-legacy-lives-on/

## Tags

go, kubernetes, gitops, continuous-delivery, cncf, helm, kustomize, cloud-native, devops, oci, cli
