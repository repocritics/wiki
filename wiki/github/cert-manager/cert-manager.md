# cert-manager/cert-manager

> Kubernetes controller that provisions, renews, and manages X.509 TLS certificates as native cluster resources.

[GitHub repo](https://github.com/cert-manager/cert-manager) ·
[Official website](https://cert-manager.io) ·
[License: Apache-2.0](https://github.com/cert-manager/cert-manager/blob/master/LICENSE)

## Overview

cert-manager is a Kubernetes add-on that turns certificate lifecycle management into declarative API objects. You describe the certificate you want (a `Certificate` resource) and where it should come from (an `Issuer` or `ClusterIssuer`), and a controller obtains it, stores it in a `Secret`, and renews it before expiry — removing the manual toil of ACME registration, CSR handling, and rotation. It supports issuance from Let's Encrypt and other ACME CAs, HashiCorp Vault, private/self-signed CAs, and commercial backends (Venafi, now marketed as CyberArk Certificate Manager)[^1].

The project began at Jetstack around 2017, loosely descended from kube-lego, and was donated to the CNCF. Jetstack was acquired by Venafi in 2020 and Venafi by CyberArk in 2024; the project itself is a CNCF graduated project and vendor-governed under that umbrella rather than a single company[^2]. The import path moved from `github.com/jetstack/cert-manager` to `github.com/cert-manager/cert-manager` at v1.8[^1].

The defining tension is that cert-manager inserts itself into the Kubernetes admission path. Its webhook must be reachable for the API server to accept any `cert-manager.io` object, and its correctness depends on external systems (ACME servers, DNS providers, CA APIs) that fail in ways Kubernetes cannot retry away. It is close to mandatory infrastructure in most clusters, and its failure modes are correspondingly load-bearing.

## Getting Started

Install via Helm, including the CRDs:

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

Define a cluster-wide ACME issuer using Let's Encrypt with an HTTP-01 solver:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

Request a certificate; cert-manager writes the signed cert + key into `example-com-tls`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - www.example.com
```

For the common Ingress case you can skip the `Certificate` object entirely and annotate the Ingress with `cert-manager.io/cluster-issuer: letsencrypt-prod`; the ingress-shim controller creates the `Certificate` for you.

## Architecture / How It Works

cert-manager ships as three separate Deployments, each a distinct failure domain:

1. **controller** — the reconcile loop that watches the CRDs and drives issuance.
2. **webhook** — a validating/mutating admission webhook and CRD conversion webhook. The API server calls it on every write to `cert-manager.io` resources.
3. **cainjector** — injects CA bundles into the webhook's own `ValidatingWebhookConfiguration`/`APIService` objects and, optionally, into user resources.

The resource model is a pipeline of CRDs. A `Certificate` reconciles into a `CertificateRequest` (a single point-in-time signing request). For ACME issuers the request spawns an `Order`, which spawns one `Challenge` per identifier; the controller solves each challenge (HTTP-01 by serving a token via a temporary Ingress/Pod, or DNS-01 by writing a `_acme-challenge` TXT record through a provider solver), then finalizes the order and stores the certificate. `Issuer` (namespaced) and `ClusterIssuer` (cluster-scoped) hold the credentials and configuration for a signing backend.

Renewal is time-driven: by default cert-manager renews at roughly two-thirds of the certificate's lifetime, or per an explicit `renewBefore`. Secrets referenced by a `ClusterIssuer` live in cert-manager's own namespace (the "cluster resource namespace"); a namespaced `Issuer` reads its secrets from the same namespace as the issuer. This scoping catches many first-time users.

The webhook's dual role — admission *and* version conversion — means the CRDs cannot be served, and existing objects cannot be read, if every webhook replica is down. This is the single most important architectural fact for operators.

## Production Notes

**The webhook is a chokepoint.** Because the API server must reach the webhook to admit `cert-manager.io` objects, a fully-unavailable webhook breaks not just issuance but any `kubectl apply` of these resources. Run multiple webhook replicas, spread across nodes, and be aware of the bootstrap chicken-and-egg: cert-manager's own serving cert (managed by cainjector) must come up before the webhook is trusted.

**ACME rate limits are real and unforgiving.** Let's Encrypt production caps certificates per registered domain per week and limits duplicate certificates[^3]. A misconfigured issuer that loops on failures can exhaust the weekly budget quickly. Always validate against the staging endpoint (`acme-staging-v02.api.letsencrypt.org`) first; staging certs are untrusted but the flow is identical.

**DNS-01 is where wildcard and private-DNS setups go wrong.** cert-manager performs its own recursive propagation self-check before asking the CA to validate. Split-horizon DNS, slow provider propagation, and CNAME delegation all cause challenges to hang in `pending`. The `--dns01-recursive-nameservers` and `--dns01-recursive-nameservers-only` controller flags exist specifically for these cases.

**cainjector memory.** Historically cainjector cached broad object sets and could dominate cert-manager's memory footprint on large clusters; recent versions expose flags to narrow what it watches. Budget for it separately from the controller.

**Upgrades touch CRDs and stored API versions.** Helm does not upgrade CRDs installed as raw manifests, and older cert-manager releases removed long-deprecated API versions (`v1alpha2`, `v1alpha3`, `v1beta1`) that had to be migrated off before upgrading[^4]. Read the release notes for every minor bump; use `cmctl` to check status and convert resources. Do not skip minors blindly.

**No Go module compatibility guarantee.** Most code under `pkg/` may change in a breaking way even between patch releases[^1]. Importing cert-manager as a library is at your own risk; the CRD API versions are the only stable contract, governed by the Kubernetes deprecation policy.

## When to Use / When Not

**Use when:**
- You run Kubernetes and want declarative, auto-renewing TLS for Ingress/Gateway workloads.
- You need ACME (Let's Encrypt) automation, or a uniform abstraction over multiple CA backends.
- You want certificate state to be a first-class, auditable cluster resource rather than scripts and cron.

**Avoid when:**
- You are not on Kubernetes — a reverse proxy with built-in ACME (Caddy, Traefik) is far simpler.
- You need a full internal PKI / CA rather than an issuance orchestrator; run a dedicated CA instead.
- Your cluster cannot tolerate an extra admission webhook in the critical path, or you cannot commit to the upgrade discipline the CRDs demand.

## Alternatives

- smallstep/certificates — run your own ACME-capable CA (step-ca); use when you want to *be* the certificate authority for internal PKI, not just orchestrate issuance.
- caddyserver/caddy — automatic HTTPS at the proxy layer; use for single-server or non-Kubernetes deployments where CRDs are overkill.
- traefik/traefik — ingress/proxy with built-in ACME; use when Traefik is already your edge and you don't need the multi-issuer abstraction.
- hashicorp/vault — PKI secrets engine as a CA backend; use directly (with Vault Agent) for non-Kubernetes workloads, or behind cert-manager's Vault issuer.
- cert-manager/trust-manager — sibling project for distributing CA trust bundles; complements rather than replaces cert-manager.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017 | Early Jetstack releases, descended from kube-lego[^2]. |
| 1.0 | 2020-09 | Stable `cert-manager.io/v1` API GA[^5]. |
| 1.6 | 2021-10 | Removed deprecated `v1alpha2`/`v1alpha3`/`v1beta1` API versions[^4]. |
| 1.8 | 2022 | Import path moved `jetstack/` → `cert-manager/`[^1]. |

## References

[^1]: cert-manager README, "Importing cert-manager as a Module" and module path notes. https://github.com/cert-manager/cert-manager
[^2]: cert-manager docs, project history and CNCF status. https://cert-manager.io/docs/
[^3]: Let's Encrypt, "Rate Limits." https://letsencrypt.org/docs/rate-limits/
[^4]: cert-manager docs, "Upgrading" and removed API versions. https://cert-manager.io/docs/releases/
[^5]: cert-manager release notes. https://cert-manager.io/docs/release-notes/
## Tags

kubernetes, tls, certificates, acme, letsencrypt, pki, go, crd, cloud-native, cncf, x509, controller
