# external-secrets/external-secrets

> A Kubernetes operator that reads secrets from external managers (Vault, AWS/GCP/Azure, and dozens more) and syncs them into native Kubernetes Secrets via CRDs.

[GitHub repo](https://github.com/external-secrets/external-secrets) ·
[Official website](https://external-secrets.io) ·
[License: Apache-2.0](https://github.com/external-secrets/external-secrets/blob/main/LICENSE)

## Overview

External Secrets Operator (ESO) is a Kubernetes controller that bridges third-party
secret stores and the cluster's own Secret objects. You declare an `ExternalSecret`
resource pointing at a backend (AWS Secrets Manager, HashiCorp Vault, GCP Secret
Manager, Azure Key Vault, and ~30 others), and the operator fetches the values and
materializes them as a standard `Secret` that pods consume the ordinary way — env
vars, volume mounts, `imagePullSecrets`. Nothing downstream has to know the secret
came from outside the cluster.

The project is a consolidation. It grew out of GoDaddy's Node.js
`kubernetes-external-secrets` and several parallel efforts, which merged in 2020 into
a single Go rewrite maintained under a vendor-neutral org[^1]. It is a CNCF project[^2],
and one of the standard answers to "how do I get secrets into Kubernetes without
committing them to Git." As of 2026 it remains pre-1.0 (v0.x), which is the defining
tension: it is widely deployed in production yet still reserves the right to break its
CRD API between minor releases.

The honest framing: ESO does not make your cluster more secure by itself. It moves the
source of truth to a real secret manager, but the synced value still lands in etcd as a
base64 `Secret`, subject to the same exposure as any other Kubernetes Secret. Its value
is operational (single source of truth, rotation, GitOps-friendly manifests), not a
reduction of the etcd threat surface.

## Getting Started

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

```yaml
# A SecretStore names a backend + how to authenticate to it (namespaced).
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-store
  namespace: app
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef: { name: eso-sa }   # IRSA
---
# An ExternalSecret maps backend keys to a target Kubernetes Secret.
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: app
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-store, kind: SecretStore }
  target:
    name: db-credentials          # the Secret ESO will create/own
  data:
    - secretKey: password          # key in the resulting Secret
      remoteRef:
        key: prod/db               # path in AWS Secrets Manager
        property: password
```

## Architecture / How It Works

ESO is a set of controllers over a handful of CRDs:

- **`SecretStore` / `ClusterSecretStore`** — a backend definition plus auth config.
  The namespaced form is per-team; the cluster-scoped form is shared but requires you
  to think carefully about which namespaces may reference it.
- **`ExternalSecret` / `ClusterExternalSecret`** — the desired mapping from remote keys
  to a target `Secret`. `dataFrom` pulls an entire remote secret; `data` cherry-picks.
- **`PushSecret`** — the reverse direction: take an existing Kubernetes Secret and write
  it *out* to a provider.
- **Generators** — dynamic values that aren't stored anywhere (e.g. `Password`,
  `ECRAuthorizationToken`, `GCRAccessToken`, ACR tokens) produced at reconcile time.

The reconcile loop is poll-based. Each `ExternalSecret` re-fetches on its own
`refreshInterval`, runs the fetched values through an optional Go template engine (used
to reshape provider output, base64-decode, build `.dockerconfigjson`, etc.), and writes
an owned `Secret`. Provider support is a plugin surface: every backend implements a
common `Provider` interface, which is why the list is long and uneven in maturity.

There is no event stream from the backends — ESO does not know a secret rotated until it
next polls. Freshness is bounded by `refreshInterval`, not by rotation events (a few
providers can do better, but the polling model is the baseline mental model).

## Production Notes

**Polling has a cost.** Every `ExternalSecret` independently calls its backend on each
interval. With thousands of resources and a short `refreshInterval`, you generate a large,
constant API call volume — this hits provider rate limits and, on AWS Secrets Manager
(billed per 10k API calls), shows up on the bill. Tune intervals per criticality; don't
default everything to `1m`.

**Pods don't reload on their own.** ESO updates the `Secret`, but a Pod that mounted it
as env vars keeps the old values until restarted. Teams pair ESO with a reloader
(e.g. stakater/Reloader) or roll deployments on secret change. Volume-mounted secrets do
update in place, with kubelet's sync delay.

**CRD upgrades are a manual footgun.** Helm does not upgrade CRDs on `helm upgrade` by
default. Skipping the documented CRD-apply step on version bumps is the most common way
people break their ESO install. The API has also moved (`v1alpha1` → `v1beta1` → `v1`);
plan conversions rather than assuming old manifests keep validating.

**It doesn't shrink the etcd threat model.** The synced `Secret` lives in etcd like any
other. If your goal was "secrets never at rest in the cluster," ESO is the wrong tool —
look at the Secrets Store CSI Driver, which can mount without persisting a `Secret`.

**Auth is where the time goes.** Most real incidents are not ESO bugs but backend IAM:
IRSA/Workload Identity/Managed Identity misconfig, over-broad `ClusterSecretStore` reach
across tenants, or a `ServiceAccount` that can read more than it should. Treat the
`SecretStore` as a privilege boundary and scope it tightly.

**Pre-1.0 cadence.** Being v0.x, minor releases can carry breaking changes. Pin the chart
version, read release notes before upgrading, and don't track `latest`.

## When to Use / When Not

**Use when:**
- You already run a real secret manager and want its values in the cluster without
  copying secrets into Git or CI.
- You want a uniform, declarative interface across AWS/GCP/Azure/Vault/etc.
- You need rotation, `PushSecret`, or generated credentials expressed as Kubernetes manifests.

**Avoid when:**
- You want secrets never materialized as a Kubernetes `Secret` in etcd — use a CSI-driver
  mount instead.
- Your only backend is Vault and you want tighter, Vault-native lifecycle semantics.
- You want encrypted-secrets-in-Git with no external service to run — SOPS/sealed-secrets
  fit that model better.
- You need a strict, stable API contract today — the pre-1.0 status may not suit you.

## Alternatives

- kubernetes-sigs/secrets-store-csi-driver — mounts provider secrets as volumes; use it
  when you specifically want to avoid syncing values into etcd `Secret` objects.
- hashicorp/vault-secrets-operator — HashiCorp's own operator; use it if you are all-in
  on Vault and want first-party lifecycle handling.
- bitnami-labs/sealed-secrets — encrypt secrets and commit them to Git; use it when you
  want a GitOps flow with no external secret manager to operate.
- getsops/sops — file-level encryption (often with Flux/Argo); use it when secrets should
  live in Git, encrypted, rather than in an external store.
- infisical/infisical — full secret-management platform with its own K8s operator; use it
  when you want the store and the sync from one vendor.

## History

| Version | Date | Notes |
|---------|------|-------|
| kubernetes-external-secrets | ~2019 | GoDaddy's original Node.js implementation (now deprecated)[^1]. |
| ESO org / Go rewrite | 2020-11 | Repo created; merger of competing projects into one Go operator[^1]. |
| v0.1.0 | 2021 | First tagged release of the rewritten operator. |
| CNCF acceptance | 2022 | Accepted into the CNCF[^2]. |
| v0.x (current) | 2026 | Still pre-1.0; CRD API stabilized to `external-secrets.io/v1`, ~30+ providers. |

## References

[^1]: Origins discussion and consolidation of the External Secrets projects. https://github.com/external-secrets/kubernetes-external-secrets/issues/47
[^2]: External Secrets Operator documentation and CNCF context. https://external-secrets.io
[^3]: Provider list and API reference. https://external-secrets.io/latest/provider/aws-secrets-manager/
[^4]: Stability and support policy. https://external-secrets.io/latest/introduction/stability-support/

## Tags

kubernetes, go, secrets-management, operator, cncf, gitops, aws-secrets-manager, hashicorp-vault, cloud-native, crd
