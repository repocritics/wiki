# GoogleContainerTools/skaffold

> A client-side CLI that runs the build → push → deploy loop for Kubernetes apps, so a source edit becomes a redeployed pod without you scripting the plumbing.

[GitHub repo](https://github.com/GoogleContainerTools/skaffold) ·
[Official website](https://skaffold.dev/) ·
[License: Apache-2.0](https://github.com/GoogleContainerTools/skaffold/blob/main/LICENSE)

## Overview

Skaffold is a Google-authored command-line tool that automates the inner development loop for Kubernetes applications[^1]. You describe your project once in a `skaffold.yaml` — which images to build, how to build them, and how to deploy the resulting manifests — and Skaffold watches your source, rebuilds the changed artifacts, retags them, redeploys, streams logs, and forwards ports. It first appeared in 2018 and reached a 1.0 GA in late 2019; as of 2026 it remains actively maintained (commits land regularly on `main`) with ~15.9k stars.

The defining design choice is that Skaffold is **client-side only**: it installs nothing in your cluster and holds no persistent controller. Everything happens as a sequence of `kubectl`/`helm`/`kustomize` invocations and image builds driven from your laptop or CI runner. This keeps the operational surface small — nothing to upgrade or secure cluster-side — but it also means Skaffold is fundamentally a *workflow orchestrator over existing tools*, not a runtime. It has no opinion of its own about how your app runs; it only sequences the tools you already use.

The persistent tension in Skaffold is scope. It sits between "just run `kubectl apply` in a shell loop" and heavier platforms like Tilt or Garden that provide a UI, a service graph, and richer live-update semantics. Skaffold deliberately stays a thin, declarative pipeline. That makes it portable and CI-friendly (`skaffold run` end-to-end, or individual phases as CI/CD building blocks; `skaffold render` emits hydrated manifests for GitOps) but leaves the interactive experience comparatively bare.

## Getting Started

```bash
# macOS (Homebrew); see skaffold.dev/docs/install for Linux/Windows binaries
brew install skaffold

# Generate a skaffold.yaml by inspecting your project
skaffold init
```

A minimal `skaffold.yaml` for a single Docker image deployed with raw manifests:

```yaml
apiVersion: skaffold/v4beta6
kind: Config
build:
  artifacts:
    - image: my-app          # tagged + pushed automatically
      docker:
        dockerfile: Dockerfile
manifests:
  rawYaml:
    - k8s/deployment.yaml
deploy:
  kubectl: {}
```

```bash
# Watch loop: rebuild + redeploy on every save, stream logs, forward ports
skaffold dev

# One-shot build-and-deploy (CI, or a throwaway env)
skaffold run --default-repo gcr.io/my-project
```

`--default-repo` rewrites every image reference to your registry, which is what makes the same config work against a local kind cluster and a remote GKE cluster without editing image names.

## Architecture / How It Works

A Skaffold run is an ordered pipeline of discrete phases, each delegating to an external tool:

1. **Build** — one or more *artifacts*, each with a pluggable builder: local Docker daemon, in-cluster Kaniko, Google Cloud Build, Bazel, Jib (Maven/Gradle), Cloud Native Buildpacks, `ko` (Go), or a `custom` script. Builds are cached by a content hash of the source and build config, so unchanged artifacts are skipped.
2. **Tag** — a tag policy stamps the image: `gitCommit` (default), `sha256` (content digest), `dateTime`, `envTemplate`, or `customTemplate`.
3. **Test** — optional structure tests / custom test commands run against built images.
4. **Render** — manifests are hydrated: image placeholders are replaced with the freshly tagged references via `kubectl`, Helm, or Kustomize renderers.
5. **Deploy** — the hydrated manifests are applied, then Skaffold runs a **status check**, polling rollout status until deployments stabilize (or a timeout fires).
6. **Port-forward + log aggregation** — in `dev`/`debug`, ports are forwarded to localhost and per-container logs are multiplexed with color-coded prefixes.

The `dev` loop wraps this: a file watcher (fsnotify, with a polling fallback) maps changed files to affected artifacts using each builder's dependency detection, rebuilds only those, and redeploys. **File sync** is the important optimization — for asset or interpreted-language changes, matched files can be copied straight into running containers, skipping the rebuild/redeploy entirely (`manual`, `infer`, or `auto` sync modes; `auto` is available for Jib and Buildpacks).

`skaffold debug` is a distinct mode: it detects the container runtime (JVM, Go/Delve, Node inspector, Python debugpy) and rewrites entrypoints and container config to expose a debug port, wiring it through to your IDE.

The config itself is versioned by an `apiVersion` schema (`skaffold/v4beta*` in recent releases). Skaffold has churned through many schema versions; `skaffold fix` upgrades an older `skaffold.yaml` to the current schema. Skaffold 2.0 restructured the old monolithic `deploy` stage into the explicit **render vs deploy** split shown above, so older tutorials using `deploy.kubectl.manifests` describe a pre-2.0 schema.

## Production Notes

- **It is a dev-loop tool, not a CD system.** For production delivery, most teams use `skaffold render` to emit hydrated manifests and hand them to Argo CD or Flux. Using `skaffold run`/`deploy` as your production apply step is possible but means your release process depends on a developer-oriented CLI and its schema stability.
- **Schema churn is real.** Config `apiVersion`s are deprecated and eventually removed on a published deprecation policy. Pinning a Skaffold version in CI and running `skaffold fix` deliberately (rather than letting a floating install silently reject an old schema) avoids surprise breakage.
- **The build cache can mislead.** The content-hash cache is aggressive; if a build depends on inputs Skaffold doesn't track (a base image tag that moved, a network-fetched dependency), it may reuse a stale image. `skaffold build --cache-artifacts=false` or clearing the cache is the escape hatch.
- **Remote clusters need a reachable registry.** Local Docker builds must be pushed somewhere the cluster can pull from; `--default-repo` plus registry credentials are mandatory for anything but a local cluster with `--load` semantics.
- **Helm deployments are the classic footgun.** Passing freshly built image tags into Helm values (historically `artifactOverrides`, now `setValueTemplates`/values wiring) is fiddly and changed across the v1→v2 migration; mismatched value paths silently deploy the chart's default image instead of your build.
- **Status check timeouts.** On slow-starting workloads the deploy stabilization poll can time out and report failure even though the rollout eventually succeeds; `statusCheckDeadlineSeconds` needs tuning for heavy apps.
- **File sync is narrow.** It only helps when the changed file can be copied into a running container without a rebuild (static assets, interpreted source). Compiled languages fall back to a full rebuild/redeploy on every change unless you engineer around it.

## When to Use / When Not

**Use when:**
- You develop against Kubernetes daily and want save-to-redeploy without hand-writing build/push/apply scripts.
- You want one declarative config that runs identically on a local cluster, a teammate's machine (`git clone && skaffold dev`), and CI.
- You need `render`-based hydrated manifests to feed a GitOps tool.
- You want to avoid installing any cluster-side agent or controller.

**Avoid when:**
- You want a rich interactive UI, a live service dependency graph, or programmable dev logic — Tilt and Garden fit better.
- Your app isn't containerized / on Kubernetes; Skaffold's whole model assumes both.
- You want a complete CD/release system — Skaffold is the inner loop, not the delivery pipeline.
- You need a stable, rarely-changing config format; the schema evolves and periodically forces `skaffold fix`.

## Alternatives

- tilt-dev/tilt — use instead when you want a programmable dev loop (Starlark `Tiltfile`), fast live-update, and a web UI showing every service's status.
- garden-io/garden — use instead when you need a full stack graph spanning build, deploy, test, and multiple environments, not just the inner loop.
- telepresenceio/telepresence — use instead when you'd rather run a service as a local process intercepting real cluster traffic than rebuild images in-cluster.
- devspace-sh/devspace — use instead when you want a comparable dev loop with a more interactive, terminal-UI-driven experience.
- okteto/okteto — use instead when you want the dev container to live in the cluster (remote dev environments) rather than building locally.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-03 | Initial public release; watch-build-deploy loop for Kubernetes[^1]. |
| 1.0.0 | 2019-11 | General availability, declared production-ready[^2]. |
| 2.0.0 | 2023 | Render/deploy split, Cloud Native Buildpacks GA, schema restructuring[^3]. |

## References

[^1]: Skaffold documentation and project site. https://skaffold.dev/docs/
[^2]: Skaffold GitHub Releases (version history and dates). https://github.com/GoogleContainerTools/skaffold/releases
[^3]: Skaffold deprecation and schema policy. https://skaffold.dev/docs/references/deprecation/

## Tags

kubernetes, developer-tools, docker, containers, go, cli, ci-cd, build-tool, gitops, inner-loop
