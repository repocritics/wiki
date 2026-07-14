# crossplane/crossplane

> A framework for building control planes on Kubernetes — declare cloud infrastructure as custom resources and let controllers continuously reconcile it.

[GitHub repo](https://github.com/crossplane/crossplane) ·
[Official website](https://crossplane.io) ·
[License: Apache-2.0](https://github.com/crossplane/crossplane/blob/main/LICENSE)

## Overview

Crossplane extends the Kubernetes API server so that external resources — cloud
buckets, databases, DNS zones, SaaS accounts — become Kubernetes objects,
reconciled by controllers the same way a Deployment reconciles Pods[^1]. Instead
of running `terraform apply` from CI, you `kubectl apply` a manifest and a
long-lived controller drives the external system toward that state and keeps it
there. It is a CNCF project, incubating since 2021[^2], and one of the most
established implementations of the "control plane, not a CLI run" model of
infrastructure management.

The defining tension is continuous reconciliation versus imperative runs. Because
Crossplane owns the resource forever, it corrects drift automatically — someone
clicking in the AWS console gets reverted — but it also means every managed
resource is a live controller loop against a cloud API, with the state of record
living in etcd rather than a state file. That is the whole value proposition and
the whole operational cost: you are running, patching, and scaling a distributed
system, not invoking a binary.

The second tension is abstraction. Crossplane's headline feature, Compositions,
lets a platform team publish their own high-level APIs (an `XR` such as
`PostgreSQLInstance`) that expand into many low-level managed resources. This is
powerful for building internal platforms, but the composition machinery has been
rewritten more than once, and the v1-to-v2 transition (2025) reshaped core
concepts including how composite resources are scoped[^3].

## Getting Started

Crossplane installs into an existing Kubernetes cluster via Helm; it is not a
standalone binary.

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace
```

Install a provider (a package that adds CRDs + a controller for an external API):

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1
```

Once the provider is healthy, a managed resource is just a Kubernetes object
(schema is provider-specific; this is illustrative):

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: example-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: default
```

`kubectl apply` it, then `kubectl get bucket` shows `SYNCED`/`READY` columns that
track the real external resource.

## Architecture / How It Works

Crossplane is a set of controllers plus an extensible type system layered on the
Kubernetes API:

- **Providers** — OCI packages that, when installed, register CRDs and run a
  controller pod. A provider maps one external API surface. Historically these
  were monolithic (all of AWS in one provider); the ecosystem moved to *provider
  families*, splitting a cloud into service-scoped providers to reduce footprint[^4].
- **Managed Resources (MRs)** — the leaf CRDs a provider installs. Each MR
  represents exactly one external object and is reconciled continuously.
- **Compositions & CompositeResourceDefinitions (XRDs)** — an XRD defines a new
  API (its schema); a Composition defines how an instance of that API expands into
  a set of MRs. This is how platform teams offer opinionated abstractions.
- **Composition Functions** — the current mechanism for generating the resource
  set. Rather than declarative patch-and-transform, a pipeline of function
  containers (written in Go, Python, KCL, and others) computes the desired
  resources programmatically[^5]. This replaced the older native patching model as
  the recommended path.
- **Upjet** — a code-generation framework that builds Crossplane providers
  directly from Terraform providers, which is how the large Upbound provider
  catalog is maintained[^6].

Because every provider ships its full CRD set, a large provider install can add
hundreds or thousands of CRDs to the API server. State is not a file: the desired
state is the Kubernetes object, and the observed state is fetched from the cloud
each reconcile. There is no plan/apply separation — reconciliation is always on.

## Production Notes

**CRD and memory footprint is the classic footgun.** Installing a monolithic
provider registers every resource type it supports as a CRD, inflating API server
memory and etcd, and the provider controller loads schemas for all of them.
Provider families exist specifically to let you install only `provider-aws-s3`
instead of all of AWS[^4]. Audit CRD count before and after any provider install.

**Continuous reconciliation fights humans and other tools.** Crossplane will
revert out-of-band changes to resources it manages. This is desirable for drift
control but surprises teams that also click in consoles or run other IaC against
the same resources. Establish clear ownership boundaries per resource.

**Deletion and orphaning need care.** Deleting an XR cascades to its MRs and can
delete real infrastructure; conversely, misconfigured deletion policies or
finalizer removal can orphan cloud resources that keep billing. There is an entire
SIG dedicated to deletion ordering. Test teardown paths deliberately.

**Cloud API rate limits and reconcile intervals.** Thousands of MRs mean thousands
of periodic reconciles hitting provider APIs; you will hit throttling. Tune
`--poll-interval` and `--max-reconcile-rate`, and expect to shard or scale
provider pods for large fleets.

**The v1 → v2 upgrade is a real migration, not a bump.** v2 (2025) changed core
concepts — notably composite resources becoming namespaced and the deprecation of
the separate Claim type — so upgrading is a deliberate project with its own
migration guidance and a dedicated `sig-v2-migration` group[^3]. v1.20 is the
final v1 minor and receives extended, critical-fix-only support[^7].

**Connection secrets.** Provider-generated credentials land in Kubernetes Secrets
by default; for production, wire the secret-store integration (e.g. Vault via the
External Secrets ecosystem) rather than leaving credentials in etcd plaintext.

## When to Use / When Not

**Use when:**
- You are building an internal platform and want to publish your own
  self-service infrastructure APIs to app teams via `kubectl`.
- You already run Kubernetes and want drift correction and continuous
  reconciliation, not one-shot applies.
- You want a single control plane spanning multiple clouds and SaaS providers
  under one RBAC and audit model.

**Avoid when:**
- You just need to stand up infrastructure occasionally — Terraform or Pulumi are
  far less operational overhead than running a control plane.
- You do not want to run and scale Kubernetes controllers as a dependency of your
  provisioning.
- Your team is not ready to own CRD sprawl, provider upgrades, and reconcile
  tuning as ongoing work.

## Alternatives

- hashicorp/terraform — imperative plan/apply with a state file; simpler to start,
  no always-on reconciliation. Use instead when you want runs, not a control plane.
- pulumi/pulumi — infrastructure in general-purpose languages; use when you prefer
  real code and SDKs over declarative CRDs.
- aws-controllers-k8s/community — AWS's own Kubernetes controllers (ACK); use when
  you are AWS-only and want first-party CRDs without Crossplane's composition layer.
- GoogleCloudPlatform/k8s-config-connector — Google's equivalent for GCP; use when
  you are GCP-only and want Google-maintained resource controllers.
- kubernetes-sigs/cluster-api — use when the thing you are provisioning is
  Kubernetes clusters specifically, not general cloud infrastructure.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-09 | Project started at Upbound; repo created[^1]. |
| Incubating | 2021 | Promoted to CNCF incubation[^2]. |
| 1.20 | 2025-05-21 | Final v1 minor; extended critical-fix support[^7]. |
| 2.0 | 2025 | Major redesign: namespaced composite resources, Claim deprecation[^3]. |
| 2.1 | 2025-11-05 | First v2 minor after GA[^7]. |
| 2.2 | 2026-02-18 | Release-cadence minor[^7]. |
| 2.3 | 2026-05-21 | Current maintained release at time of writing[^7]. |

## References

[^1]: Crossplane README and repository metadata. https://github.com/crossplane/crossplane
[^2]: CNCF project page — Crossplane. https://www.cncf.io/projects/crossplane/
[^3]: Crossplane v2 concepts and migration (composite resources, Claim changes). https://docs.crossplane.io/latest/
[^4]: Crossplane docs — providers and provider families. https://docs.crossplane.io/latest/concepts/providers/
[^5]: Crossplane docs — Composition Functions. https://docs.crossplane.io/latest/concepts/compositions/
[^6]: Upjet — generate Crossplane providers from Terraform providers. https://github.com/crossplane/upjet
[^7]: Crossplane release cycle and maintained-release table (README). https://docs.crossplane.io/knowledge-base/guides/release-cycle

## Tags

go, kubernetes, control-plane, infrastructure-as-code, cncf, multicloud, cloud-native, operators, gitops, platform-engineering
