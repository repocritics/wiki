# helm/helm

> The package manager for Kubernetes — templated YAML bundles ("charts") installed as versioned, rollback-able releases.

[GitHub repo](https://github.com/helm/helm) ·
[Official website](https://helm.sh) ·
[License: Apache-2.0](https://github.com/helm/helm/blob/main/LICENSE)

## Overview

Helm packages a set of Kubernetes manifests into a versioned unit called a **chart**, renders that chart against user-supplied values, and tracks each install as a named **release** that can be upgraded and rolled back[^1]. It is the de facto distribution format for Kubernetes software: most vendors ship a chart, and Artifact Hub indexes tens of thousands of them[^2]. Helm is a CNCF graduated project, originally created at Deis and merged with Google's Kubernetes Deployment Manager in 2015.

The defining tension in Helm is **text templating of a structured format**. Charts are Go `text/template` (plus the Sprig function library) that emit YAML, which is then parsed into Kubernetes objects. This makes charts trivial to author and infinitely flexible, but it also means Helm manipulates whitespace-significant YAML as unstructured strings — indentation bugs, unquoted values that YAML coerces to booleans, and `{{-` chomp operators are the everyday reality of chart maintenance. Tools like Kustomize, jsonnet, and cdk8s exist largely as a reaction to this design.

The other structural fact worth knowing up front is the **v2 → v3 → v4 lineage**. Helm 2 had a privileged in-cluster server component (Tiller) that was a persistent security complaint; Helm 3 (2019) removed it entirely, making Helm a client-side tool that talks directly to the Kubernetes API with the user's own credentials[^3]. Helm v4 is the current stable line, developed on `main`; Helm v3 has moved to support-only status on the `dev-v3` branch, with bug fixes through mid-2026 and security fixes through late 2026[^4].

## Getting Started

```bash
brew install helm          # macOS; or scoop/choco/winget/snap/apt on other platforms
```

Install a public chart into your current kube-context:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-cache bitnami/redis --set architecture=standalone
helm list                              # see the release
helm upgrade my-cache bitnami/redis --set auth.enabled=true
helm rollback my-cache 1               # revert to revision 1
```

Author a chart:

```bash
helm create mychart          # scaffolds Chart.yaml, values.yaml, templates/
helm template mychart        # render locally, no cluster needed — the key debug tool
helm install web ./mychart -f prod-values.yaml
```

A minimal template pulling from `values.yaml`:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: web
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

## Architecture / How It Works

Helm 3+ is a **client-side binary with no server component**. State that used to live in Tiller now lives in the cluster as Kubernetes objects.

**Rendering pipeline.** `helm install`/`upgrade` (1) loads the chart and merges `values.yaml` with `--set`/`-f` overrides and computed built-ins (`.Release`, `.Chart`, `.Capabilities`), (2) executes every file under `templates/` as a Go template, (3) parses the resulting text into manifests, (4) sorts them by an install-order kind list (namespaces and CRDs before the workloads that use them), and (5) applies them to the API server.

**Release storage.** Each release revision is serialized, gzipped, base64-encoded, and stored as a Kubernetes **Secret** (default; configurable to ConfigMap or SQL) in the release namespace, named `sh.helm.release.v1.<name>.v<n>`[^5]. `helm history`, `rollback`, and `list` all read these objects. There is no external Helm database — the cluster *is* the source of truth.

**Upgrade diffing.** Helm computes a three-way strategic merge patch between the last-applied manifest, the newly rendered manifest, and the live cluster state, so it can detect and reconcile some out-of-band changes. This is imperfect: it does not do continuous reconciliation the way a controller would, and it can miss or clobber drift depending on field semantics.

**Charts and dependencies.** A chart is a directory (or a `.tgz`) containing `Chart.yaml`, `values.yaml`, `templates/`, and optionally sub-charts under `charts/`. Dependencies are declared in `Chart.yaml` and locked in `Chart.lock`. Since Helm 3, charts can be pushed to and pulled from **OCI registries** (`oci://`), which is now the recommended distribution path over the older `index.yaml` HTTP repositories[^6].

**Hooks.** Annotations (`helm.sh/hook: pre-install`, `post-upgrade`, etc.) let a resource run at lifecycle points — commonly Jobs for migrations. Hooks are Helm-managed, not tracked as part of the release's normal resource set, which is a frequent source of confusion.

## Production Notes

**`helm template` does not equal `helm install`.** Local rendering skips cluster-side validation, admission webhooks, `.Capabilities` API discovery, and hooks. A chart that renders cleanly can still fail on apply. Use `helm install --dry-run=server` to render against the real API server.

**YAML/whitespace footguns are the top time sink.** `indent` vs `nindent`, the `{{-`/`-}}` chomp operators, and Norway-problem coercion (`no`, `off`, `yes` becoming booleans) all bite. Always quote string values with `{{ .Values.x | quote }}`. `helm lint` and `helm template | kubectl apply --dry-run=client -f -` catch a subset.

**Releases are not reconciled.** Helm applies at `install`/`upgrade` time and then walks away. If someone `kubectl edit`s a resource, or a controller mutates it, Helm will not correct it until the next upgrade — and the three-way merge may or may not notice. Teams needing continuous drift correction wrap Helm in a GitOps controller (Argo CD, Flux) rather than relying on Helm alone.

**Stuck releases.** A failed upgrade can leave a release in `pending-upgrade`/`pending-install` state that blocks further operations. Recovery usually means `helm rollback`, deleting the offending release Secret, or `--force`. `helm upgrade --atomic` (auto-rollback on failure) and `--cleanup-on-fail` reduce but do not eliminate this class of incident.

**CRDs are a known weak spot.** Helm installs CRDs placed in a chart's `crds/` directory but **never upgrades or deletes them**, by design, to avoid destroying custom resources. Charts that ship CRDs as regular templates get upgrade behavior but risk data-loss ordering problems. Neither approach is fully satisfying; CRD lifecycle remains one of Helm's rough edges.

**Secrets in values.** Rendered manifests — including plaintext secrets — are stored in the release object in-cluster. Anyone with read access to Secrets in the namespace can read prior release payloads. Pair Helm with SOPS, Sealed Secrets, External Secrets, or `helm-secrets` for sensitive values.

**Chart quality varies wildly.** Because charts are just templates, a popular community chart may expose hundreds of `values.yaml` knobs, lag upstream image versions, or bake in opinions you must override. Read the chart's templates before trusting it in production.

## When to Use / When Not

**Use when:**
- You're distributing or consuming Kubernetes apps as installable, versioned packages.
- You need parameterized manifests across environments (dev/staging/prod values files).
- You want install/upgrade/rollback history tracked in-cluster without extra infrastructure.
- Your CI/CD or GitOps tool (Argo CD, Flux) consumes charts as its rendering step.

**Avoid when:**
- You want continuous reconciliation and drift correction — that's a controller's job, not Helm's.
- Your manifests are small and stable; raw YAML + Kustomize overlays may be simpler and avoid string-templated YAML entirely.
- You want type-safe, programmatic manifest generation — cdk8s, Pulumi, or jsonnet fit better.
- The templating indirection would obscure more than it abstracts (single-team, single-environment apps).

## Alternatives

- kubernetes-sigs/kustomize — overlay-and-patch instead of templating; no string interpolation of YAML, but weaker for parameterized redistribution. Use when you own the manifests and want environment overlays.
- cdk8s-team/cdk8s — synthesize manifests from TypeScript/Python/Go. Use when you want real programming-language logic and type safety over Go templates.
- argoproj/argo-cd — GitOps CD controller that *renders* Helm charts but adds continuous reconciliation. Use when you need drift correction, not just apply-and-forget.
- fluxcd/flux2 — GitOps toolkit with a Helm controller. Use when you want Helm releases driven declaratively from Git.
- google/jsonnet + ksonnet-style tooling — data-templating language for JSON/YAML. Use when Go templates feel too weak and you want composition primitives.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2016-11 | First major release; server-side Tiller component[^3]. |
| 3.0 | 2019-11 | Tiller removed; client-only, three-way merge, chart deps in `Chart.yaml`[^3]. |
| 3.8 | 2022-01 | OCI registry support graduated to stable[^6]. |
| 4.0 | 2025–2026 | Current stable line, developed on `main`; v3 moved to support-only on `dev-v3`[^4]. |

## References

[^1]: Helm README and documentation. https://helm.sh/docs/
[^2]: Artifact Hub — Helm chart index. https://artifacthub.io/packages/search?kind=0
[^3]: Helm 3 release notes / "Helm 3 Released!" — 2019-11-13. https://helm.sh/blog/helm-3-released/
[^4]: Helm README, "Helm Development and Stable Versions" — v4 on `main`, v3 support mode on `dev-v3`. https://github.com/helm/helm
[^5]: Helm docs, "Storage backends" (release objects stored as Secrets). https://helm.sh/docs/topics/advanced/#storage-backends
[^6]: Helm docs, "Registries" — OCI-based chart distribution. https://helm.sh/docs/topics/registries/

## Tags

go, kubernetes, package-manager, cncf, charts, templating, devops, gitops, deployment, cloud-native
