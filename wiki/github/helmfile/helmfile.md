# helmfile/helmfile

> Declarative spec for deploying Helm charts — one YAML file that describes every release in a cluster, applied like Terraform for Helm.

[GitHub repo](https://github.com/helmfile/helmfile) ·
[Official website](https://helmfile.readthedocs.io) ·
[License: MIT](https://github.com/helmfile/helmfile/blob/main/LICENSE)

## Overview

Helmfile is a Go CLI that manages collections of Helm releases from a declarative state file (`helmfile.yaml`). Instead of running `helm upgrade --install` per chart with hand-managed `-f` flags, you describe repositories, releases, per-environment values, and inter-release dependencies in one version-controlled file, then run `helmfile apply`. The tool diffs the desired state against what is deployed and only syncs releases that changed[^1]. It also flattens Kustomize configs and plain manifest directories into Helm releases, and can emit all-in-one rendered manifests for ArgoCD consumption.

The project started as roboll/helmfile (Rob Boll, 2017) and became a de facto standard for CI-driven multi-chart deployment. When the original repo went unmaintained, the community moved development to the helmfile org in early 2022[^2]; the roboll repo now redirects users here. Development remains active — v1.0 and v1.1 shipped by May 2025[^1], and pushes continue as of July 2026. At 5.2k stars (plus the predecessor repo's audience) with only 26 open issues/PRs, it is a mature, tightly-triaged tool rather than a growth-stage project.

The defining tension: Helmfile is push-based CLI orchestration in an ecosystem that has largely shifted to pull-based GitOps controllers (Argo CD, Flux). Its niche today is teams who want declarative multi-release state without running a controller — CI pipelines, ephemeral environments, air-gapped clusters — or who use `helmfile template` as a rendering front-end for a GitOps tool.

## Getting Started

Helmfile delegates to the `helm` binary; `helm` and the `helm-diff` plugin must be installed (`helmfile init` installs required plugins).

```bash
brew install helmfile     # or: scoop install helmfile / pacman -S helmfile / download a release binary
helmfile init             # installs helm-diff and other required plugins
```

```yaml
# helmfile.yaml
repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts

releases:
  - name: prom
    namespace: prometheus
    chart: prometheus-community/prometheus
    set:
      - name: rbac.create
        value: false
```

```bash
helmfile diff    # show what would change
helmfile apply   # diff, then sync only releases with a non-empty diff
```

## Architecture / How It Works

Helmfile is an orchestrator, not a reimplementation: it shells out to whatever `helm` binary is on `PATH`, so Helm version upgrades do not require Helmfile upgrades[^1]. The pipeline per run:

1. **Render the state file.** `helmfile.yaml` (or `.yaml.gotmpl`) is itself a Go template with Sprig plus Helmfile-specific functions (`requiredEnv`, `exec`, `readFile`, `fetchSecretValue`). Environments (`environments:` block, `-e` flag) inject `.Values` used during this render. Sub-helmfiles (`helmfiles:` globs) and `bases:` allow layering shared defaults under per-cluster overrides.
2. **Render values files.** Files ending in `.gotmpl` referenced from `values:` get a second templating pass before being handed to Helm.
3. **Build the DAG.** `needs:` entries (`namespace/name`) order releases; independent releases run concurrently (`--concurrency`).
4. **Diff and sync.** `helmfile apply` runs `helm diff upgrade` (via the helm-diff plugin[^3]) per release and only performs `helm upgrade --install` where the diff is non-empty. `helmfile sync` skips the diff and upgrades everything.

Non-chart sources (Kustomize builds, raw manifest directories) are converted into temporary ephemeral charts by the chartify library[^4], and JSON/strategic-merge patches can be applied to any chart before install — upstream charts can be patched without forking. Secrets integrate through the helm-secrets plugin (SOPS-encrypted values files decrypted at apply time).

## Production Notes

**Removed ≠ uninstalled.** Helmfile keeps no state store of its own; the cluster's Helm release records are the state. Deleting a release entry from `helmfile.yaml` orphans the deployed release — `apply` will not prune it. The supported pattern is setting `installed: false` on the release and applying before removing the entry. Teams discover this via leftover workloads, not error messages.

**Two templating layers.** The state file and `.gotmpl` values are rendered by Helmfile's Go templating *before* Helm renders chart templates. Passing a literal `{{ }}` through to a chart requires escaping, and error messages rarely say which layer failed. This is the most common source of confusion for new users.

**helm-diff is load-bearing and imperfect.** `apply`'s decision to sync depends entirely on the plugin's diff. helm-diff compares rendered manifests against the *last deployed Helm release*, not live cluster state — drift introduced via `kubectl apply` is invisible. Pin the helm-diff version in CI: version mismatches between machines produce spurious diffs and surprise syncs.

**Performance scales with subprocess count.** Every release is one or more `helm` invocations; state files with hundreds of releases get slow. Mitigations: `--concurrency N`, `--skip-deps` to avoid re-fetching repos/chart dependencies each run, and `-l` selector labels to operate on a subset.

**v0.x → v1 migration.** The v1 line landed a deliberate set of breaking changes[^5]; the most visible per the proposal is that a plain `helmfile.yaml` is no longer rendered as a Go template — templated state files must use the `.gotmpl` extension. The maintainers recommend v0.x users upgrade straight to v1.1[^1]. Budget a rename-and-verify pass, not a drop-in swap.

**CI ergonomics are the point.** `helmfile diff` in a PR check plus `helmfile apply` on merge is the canonical setup, and `--detailed-exitcode`-style diff gating works well. But because apply runs from CI credentials with cluster-admin-ish rights, the state file's `exec` template function is an arbitrary-command surface — review templated helmfiles like code, because they are.

## When to Use / When Not

**Use when:**
- You deploy many Helm charts per cluster and want the set, versions, and values under version control with a reviewable diff.
- You need per-environment value layering (dev/stage/prod) without duplicating chart config.
- You run CI-driven (push) deployment and don't want to operate a GitOps controller.
- You want to fold Kustomize configs and raw manifests into the same release workflow, or patch upstream charts without forking.
- You feed rendered manifests to ArgoCD and want one `helmfile template` source of truth.

**Avoid when:**
- You want continuous in-cluster reconciliation and drift correction — a pull-based controller (Flux, Argo CD) does what Helmfile structurally cannot.
- You manage a handful of releases; plain Helm plus a Makefile is less machinery.
- Your Kubernetes config lives in Terraform/Pulumi already; a second orchestrator splits the plan.
- Your team is allergic to Go templating — Helmfile doubles down on it.

## Alternatives

- helm/helm — use plain Helm when you have few releases and no environment layering; Helmfile is a wrapper over it anyway.
- fluxcd/flux2 — use instead when you want pull-based GitOps: HelmRelease CRs reconciled continuously in-cluster, with drift correction.
- argoproj/argo-cd — use instead for GitOps with a UI and app-of-apps structure; Helmfile can still serve as its manifest renderer.
- hashicorp/terraform-provider-helm — use instead when Helm releases should live in the same plan/apply lifecycle as your cloud infrastructure.
- kubernetes-sigs/kustomize — use instead if you want patch-based, template-free config and don't need Helm releases at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017 | Initial release by Rob Boll (roboll/helmfile); long 0.x line maintained largely by Yusuke Kuoka (mumoshu). |
| — | 2022-03 | Development moves to the helmfile org after the original repo goes unmaintained[^2]; helmfile/helmfile created 2022-03-27. |
| 1.0 | 2025 | First stable major; breaking changes per the v1 proposal, notably `.gotmpl`-only state-file templating[^5]. |
| 1.1 | 2025 | Released by May 2025; recommended upgrade target for remaining v0.x users[^1]. |

## References

[^1]: Helmfile README — About, Status (May 2025 update), Installation. https://github.com/helmfile/helmfile#readme
[^2]: roboll/helmfile — original repository, deprecated in favor of helmfile/helmfile. https://github.com/roboll/helmfile
[^3]: helm-diff plugin (databus23/helm-diff) — required by `helmfile diff`/`apply`. https://github.com/databus23/helm-diff
[^4]: chartify — converts Kustomize/manifest directories into ephemeral Helm charts. https://github.com/helmfile/chartify
[^5]: Helmfile v1 proposal, "Towards Helmfile 1.0". https://github.com/helmfile/helmfile/blob/main/docs/proposals/towards-1.0.md

## Tags

go, kubernetes, helm, deployment, gitops, infrastructure-as-code, cli, devops, configuration-management, kustomize
