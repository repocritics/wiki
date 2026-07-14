# argoproj/argo-cd

> Declarative GitOps continuous delivery for Kubernetes — a controller that reconciles a cluster's live state against manifests stored in Git.

[GitHub repo](https://github.com/argoproj/argo-cd) ·
[Official website](https://argo-cd.readthedocs.io) ·
[License: Apache-2.0](https://github.com/argoproj/argo-cd/blob/master/LICENSE)

## Overview

Argo CD is a Kubernetes controller that implements the GitOps pattern: a Git repository holds the desired state of one or more applications (as plain YAML, Helm charts, Kustomize overlays, or Jsonnet), and Argo CD continuously compares that desired state against what is actually running in the cluster, reporting drift and optionally correcting it. It originated at Intuit (via the 2018 Applatix acquisition) and is part of the Argo project family alongside Argo Workflows, Argo Rollouts, and Argo Events[^1]. It became a CNCF graduated project in December 2022[^2].

The unit of deployment is the `Application` custom resource, which points at a repo/path/revision and a target cluster/namespace. Argo CD renders the manifests, diffs them against live objects, and surfaces the result as `Synced`/`OutOfSync` plus a health status. It ships a web UI, a CLI (`argocd`), and a gRPC/REST API — the UI is a genuine differentiator, giving operators a visual application topology and diff view that most competing tools lack.

The defining tension is **pull vs. push and where reconciliation authority lives**. Argo CD runs inside (or adjacent to) the target cluster and pulls from Git, which means Git is the audit log and the source of truth — but it also means Argo CD itself becomes a privileged, stateful control plane that you now have to operate, scale, and secure. It is heavier than a plain `kubectl apply` pipeline, and the operational surface (repo-server, application-controller, Redis, RBAC, secrets handling) is where most of the real cost shows up.

## Getting Started

Install into a dedicated namespace, then expose the API server and log in:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# initial admin password is the pod name (older) or a generated secret:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d

argocd login <ARGOCD_SERVER>
```

A minimal `Application` (this is itself a manifest you can commit — the "app of apps" pattern):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true      # delete resources removed from Git
      selfHeal: true   # revert manual cluster changes
```

## Architecture / How It Works

Argo CD is not a single binary; it is a set of cooperating components[^3]:

1. **application-controller** — the reconciliation engine. It watches `Application` resources, compares desired vs. live state, and performs syncs. This is the stateful, CPU/memory-hungry core.
2. **repo-server** — clones Git repos and renders manifests (runs `helm template`, `kustomize build`, Jsonnet, or config management plugins). Rendering is cached; this component is CPU-bound and the usual first scaling bottleneck.
3. **api-server** — gRPC/REST server backing the UI and CLI; handles auth, RBAC, and the application API.
4. **redis** — an ephemeral cache for rendered manifests and computed state. It is a cache, not a database — losing it forces re-computation but not data loss.
5. **applicationset-controller** — generates `Application` resources from generators (list, cluster, Git directory, pull-request, matrix). This is how teams template hundreds of apps across many clusters.
6. **notifications-controller** and optionally **dex** (federated SSO) round out the deployment.

State lives in Kubernetes itself: `Application`, `ApplicationSet`, and `AppProject` are CRDs, and configuration/secrets live in ConfigMaps and Secrets (`argocd-cm`, `argocd-rbac-cm`, `argocd-secret`). There is no external database.

Reconciliation is a loop: the controller polls Git (default ~3 minutes, or immediately via webhook), renders manifests, diffs, and — if `syncPolicy.automated` is set — applies. **Sync waves** and **resource hooks** (`PreSync`/`Sync`/`PostSync`) give ordered, phased rollouts; annotations control ordering. Diffing is structural and has well-known sharp edges around mutating admission webhooks, defaulted fields, and CRDs the controller does not fully understand — hence the frequent need for `ignoreDifferences` rules.

Argo CD deliberately does **not** do progressive delivery (canary/blue-green) itself; that is the separate `argoproj/argo-rollouts` project. It also does not manage secrets — it renders whatever is in Git, so encrypted-secret tooling is a bring-your-own decision.

## Production Notes

- **Scaling is a real project, not a checkbox.** The application-controller is the bottleneck at scale; running many applications or many clusters requires sharding the controller across replicas (`ARGOCD_CONTROLLER_REPLICAS` + a sharding algorithm). Cluster-per-shard imbalance is a recurring operational headache; newer dynamic sharding helps but tuning is manual.
- **repo-server is CPU-bound.** Heavy Helm/Kustomize rendering, large monorepos, and many concurrent syncs saturate it first. Scale it horizontally and raise its timeout/parallelism limits before touching anything else.
- **The UI degrades with app count.** Thousands of applications and large resource trees make the web UI and API sluggish; teams often rely on the CLI and per-project scoping past a certain scale.
- **Secrets are your problem.** Argo CD has no native secrets story. Common patterns: Sealed Secrets, External Secrets Operator, SOPS via a config management plugin, or Vault. Putting plaintext secrets in Git is the classic anti-pattern the tool does not prevent.
- **Auto-sync + self-heal can fight you.** `selfHeal: true` reverts *any* out-of-band change, including emergency `kubectl edit` fixes during an incident. `prune: true` will delete resources dropped from Git — including ones removed by mistake. Both are correct GitOps and both have caused outages.
- **Diff noise.** Admission controllers, mutating webhooks, and server-side-applied defaults cause perpetual `OutOfSync` states that require `ignoreDifferences` / server-side-apply configuration. Expect to spend time here.
- **RBAC is two-layered.** Kubernetes RBAC governs what the controller's service account can do in target clusters; Argo CD's own `argocd-rbac-cm` governs what users/roles can do in the UI/API. They are independent and both must be right.
- **Upgrade discipline.** CRD and manifest changes across minor versions occasionally require reading the upgrade notes carefully; skipping intermediate minors is discouraged. Notifications, ApplicationSet, and RBAC syntax have all shifted across releases.

## When to Use / When Not

**Use when:**
- You want Git as the single source of truth and audit trail for Kubernetes deployments.
- You run multiple clusters/environments and need a consistent, visual deployment control plane.
- You value drift detection and the ability to revert manual cluster changes automatically.
- Your team already thinks in Helm/Kustomize and wants declarative, reviewable rollouts.

**Avoid when:**
- You have a handful of services and a simple CI pipeline — `kubectl apply` or Helm in CI is lighter and you skip operating a control plane.
- You need imperative, push-style deploys tightly coupled to a CI job's exit code (Argo CD's pull model adds a reconciliation gap).
- You are not on Kubernetes. Argo CD is Kubernetes-only by design.
- You want built-in canary/progressive delivery — that is Argo Rollouts, a separate install.

## Alternatives

- fluxcd/flux2 — the other CNCF-graduated GitOps tool; more composable/controller-set-oriented and CLI-first, weaker built-in UI. Use instead when you prefer a lighter, toolkit-style GitOps footprint.
- argoproj/argo-rollouts — complementary, not competing; use alongside Argo CD when you need canary/blue-green progressive delivery.
- spinnaker/spinnaker — heavyweight multi-cloud CD with rich pipelines; use when you need complex non-Kubernetes deployment orchestration and can afford the operational weight.
- jenkins-x/jx — opinionated GitOps-on-Jenkins for full CI+CD; use when you want an all-in-one preview-environment workflow rather than a focused CD controller.
- Codefresh / Akuity — commercial platforms built on Argo CD; use when you want hosted, supported Argo CD at scale instead of self-operating it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019-05 | First stable release; core Application CRD and sync engine[^1]. |
| 2.0 | 2021-04 | ApplicationSet, notifications, pod logs in UI, major UX overhaul[^4]. |
| — | 2022-12 | Graduated within the CNCF[^2]. |
| 2.6 | 2023-02 | ApplicationSet promoted, multiple sources per Application. |
| 2.x | 2023–2024 | Server-side diff, dynamic controller sharding, progressive ApplicationSet syncs. |
| 3.0 | 2025 | Major release; tightened defaults and security-oriented changes[^5]. |

## References

[^1]: Argo project overview and history. https://argoproj.github.io/
[^2]: CNCF, "Argo Project graduates" — 2022-12-06. https://www.cncf.io/announcements/2022/12/06/argo-cd-and-argo-rollouts-graduate/
[^3]: Argo CD architecture — core components and reconciliation. https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/
[^4]: Argo CD 2.0 release notes. https://github.com/argoproj/argo-cd/releases/tag/v2.0.0
[^5]: Argo CD release history. https://github.com/argoproj/argo-cd/releases

## Tags

go, kubernetes, gitops, continuous-delivery, cd, devops, cncf, declarative, helm, kustomize, cloud-native
