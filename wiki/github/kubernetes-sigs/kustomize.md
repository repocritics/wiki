# kubernetes-sigs/kustomize

> Template-free customization of Kubernetes YAML via bases and overlays — patch, don't template.

[GitHub repo](https://github.com/kubernetes-sigs/kustomize) ·
[Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/) ·
[License: Apache-2.0](https://github.com/kubernetes-sigs/kustomize/blob/master/LICENSE)

## Overview

kustomize is a CLI for producing Kubernetes manifests by layering declarative
edits over unmodified base YAML. Instead of parameterizing manifests with a
templating language (the Helm approach), you keep plain, valid Kubernetes
objects and describe changes — label injection, name prefixes, image tag
overrides, strategic-merge patches — in a `kustomization.yaml` file. `kustomize
build` walks the graph and emits the rendered YAML to stdout[^1]. It is a
kubernetes-sig-cli project, developed under a KEP, not a Vercel-style
vendor-owned framework.

Its defining decision is "template-free." Every input file is a real,
apply-able Kubernetes resource, so tooling (schema validation, IDE completion,
`kubectl diff`) works on the sources directly. The cost is that expressiveness
is bounded: there are no loops, no conditionals, and no string interpolation.
Anything that a template would express with a variable, kustomize expresses with
a patch or a transformer, which is verbose for high-variance configuration and
sometimes impossible without escaping into plugins.

kustomize occupies an unusual position because it is also embedded in `kubectl`
itself (`kubectl apply -k`, `kubectl kustomize`)[^2]. This makes it the default
zero-install manifest tool for anyone with kubectl, but the embedded copy lags
the standalone binary by one or more minor versions, which is a recurring source
of "works on my machine" confusion.

## Getting Started

```bash
# Standalone binary (recommended — newer than the kubectl-embedded copy)
brew install kustomize
# or: go install sigs.k8s.io/kustomize/kustomize/v5@latest

kustomize version
```

A minimal base plus a production overlay:

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
labels:
  - pairs: { app: myapp }
    includeSelectors: true
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: prod-
images:
  - name: myapp
    newTag: "1.4.2"
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 8
    target: { kind: Deployment, name: myapp }
```

```bash
kustomize build overlays/prod | kubectl apply -f -
# or, with the embedded copy:
kubectl apply -k overlays/prod
```

## Architecture / How It Works

A `kustomization.yaml` is a directed graph node. Its `resources` field points at
files or at other directories that themselves contain a `kustomization.yaml`
(the base/overlay recursion). `kustomize build` loads the leaf resources, then
applies **generators** first and **transformers** second, bottom-up through the
overlay chain.

- **Generators** create resources: `configMapGenerator` and `secretGenerator`.
  By default they append a content hash suffix to the object name (e.g.
  `myapp-map-8t9fk2hd6c`). A ConfigMap change therefore produces a new name,
  which forces referencing Deployments to roll — the mechanism that makes config
  changes trigger restarts. `generatorOptions.disableNameSuffixHash: true` turns
  this off, at the cost of losing automatic rollout on config change.
- **Transformers** mutate resources: `namePrefix`/`nameSuffix`, `namespace`,
  `labels`, `images`, `replicas`, and patches. Patches come in two flavors:
  strategic-merge (partial YAML merged by Kubernetes merge-key semantics) and
  JSON 6902 (explicit `op`/`path` operations). The modern unified `patches`
  field accepts either and adds a `target` selector so one patch can hit many
  objects.

The rendering engine is [kyaml](https://github.com/kubernetes-sigs/kustomize/tree/master/kyaml),
a separate library that preserves comments and formatting by editing the YAML
node tree rather than round-tripping through Go structs. kustomize's own public
Go packages live under `sigs.k8s.io/kustomize/api` and `sigs.k8s.io/kustomize/kyaml`;
these are consumed by Argo CD, Flux, and kubectl, so kyaml behavior changes ripple
widely across the GitOps ecosystem.

Extensibility is the **KRM Functions** model (formerly "config functions" /
plugins): a transformer or generator implemented as a container or exec binary
that reads a ResourceList on stdin and writes the mutated list on stdout. This
is powerful but requires `--enable-alpha-plugins` (and `--enable-exec` /
`--network` for exec and container functions), and most teams never touch it.

**Components** (`kind: Component`) are a second composition axis: reusable,
optional overlay fragments that can be mixed into multiple overlays, addressing
the fact that plain bases compose linearly and can't express "feature X, applied
to several environments."

## Production Notes

- **Two versions, silent skew.** `kubectl kustomize` uses an embedded copy that
  trails the standalone binary. Features and bug fixes present in standalone
  kustomize may be absent in your kubectl. Pin a standalone binary in CI and
  render there; don't rely on `apply -k` for anything using recent fields[^2].
- **Load restrictions bite in CI.** By default kustomize refuses to read files
  outside the kustomization's own directory tree (a path-traversal safeguard).
  Symlinked or shared config directories fail with a "security; file is not in
  or below" error until you pass `--load-restrictor LoadRestrictionsNone`, which
  reopens the risk it was guarding against[^3].
- **The deprecation churn.** Long-lived tutorials teach fields that now warn or
  are removed. `commonLabels` → `labels`; `patchesStrategicMerge` /
  `patchesJson6902` → `patches`; `vars` → `replacements`; `bases` folded into
  `resources`. `vars` in particular never supported arbitrary substitution and
  was deprecated in favor of `replacements`, which is more capable but far more
  verbose. Copy-pasted old kustomizations are a steady source of build failures.
- **Remote bases are a supply-chain surface.** `resources` can reference a Git
  URL. Convenient, but it pulls unpinned config at build time unless you pin to a
  commit SHA. Historically kustomize used `go-getter` for this; remote fetching
  has been tightened over time for security. Treat remote bases like remote code.
- **Name-hash suffixes surprise people.** The generator hash suffix means the
  emitted ConfigMap/Secret name is not the name you wrote. References that
  kustomize itself rewrites (via `valueFrom`, env, volumes) are updated
  automatically; references from outside the kustomization (another tool, a raw
  manifest) are not, and break.
- **No dry-run of intent, only of output.** kustomize renders text; it has no
  knowledge of live cluster state. `kustomize build | kubectl diff -f -` is the
  real "what will change" check. Ordering of emitted resources is deterministic
  but not necessarily apply-safe ordering (CRDs before CRs, etc.) — that is
  kubectl's problem, not kustomize's.
- **Helm inflation is a bridge, not a home.** `--enable-helm` lets a
  kustomization inflate a Helm chart and then patch it, but chart templating runs
  as a subprocess with its own flags and version constraints, and the combination
  is fragile. It is best used to patch a third-party chart you don't own, not as
  a primary workflow.

## When to Use / When Not

**Use when:**
- You want manifests that stay valid Kubernetes YAML and play well with schema
  validation, `kubectl diff`, and GitOps controllers (Argo CD / Flux both
  render kustomizations natively).
- Your variance is structural — a handful of environments differing by replicas,
  images, labels, namespaces — rather than combinatorial parameterization.
- You already have kubectl and want zero additional install for basic cases.

**Avoid when:**
- Your configuration is highly parametric (dozens of tunable values, conditional
  blocks, loops). That is Helm's or a real programming language's job; expressing
  it as patches is painful.
- You need packaging, versioning, and distribution of an app to third parties.
  kustomize has no release/registry story; Helm charts do.
- You want a single source of truth with rich typing and reuse — a
  general-purpose config language (jsonnet, ytt, cdk8s) fits better.

## Alternatives

- helm/helm — use instead when you need templating, packaging, versioned
  releases, and distribution of an app to other people.
- carvel-dev/ytt — use instead when you want real templating/overlays with a
  Starlark-based language but still prefer a YAML-native tool.
- cdk8s-team/cdk8s — use instead when you'd rather define manifests in
  TypeScript/Python/Go with full language power than in YAML patches.
- kubernetes-sigs/kpt — use instead when you want "configuration as data" with
  KRM function pipelines as the primary model rather than an add-on.
- google/jsonnet — use instead when you want a single data-templating language
  across all your config, not just Kubernetes.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.x | 2018 | Initial standalone releases under sig-cli. |
| v2.0.3 | 2019-03 | Version embedded into kubectl v1.14 as `apply -k`[^2]. |
| v3.0.0 | 2019 | Expanded transformer set and plugin framework. |
| v4.0.0 | 2021 | Go module restructure; embedded copy updated to v4.0.5 in kubectl v1.21. |
| v5.0.0 | 2023 | Removed several long-deprecated fields/flags; `labels`/`patches`/`replacements` established as the preferred forms. |

## References

[^1]: kustomize README and glossary — kubernetes-sigs/kustomize. https://github.com/kubernetes-sigs/kustomize
[^2]: "Kustomize integration" and kubectl version table — kustomize README; kubectl v1.14 release announcement. https://kubernetes.io/blog/2019/03/25/kubernetes-1-14-release-announcement
[^3]: kustomize load restrictor documentation. https://kubectl.docs.kubernetes.io/references/kustomize/

## Tags

go, kubernetes, yaml, configuration-management, gitops, kubectl, devops, cli, sig-cli, manifests, overlays
