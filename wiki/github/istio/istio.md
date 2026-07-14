# istio/istio

> The reference service mesh: a control plane that programs Envoy sidecars (and now sidecar-less proxies) to secure, route, and observe traffic between services.

[GitHub repo](https://github.com/istio/istio) ·
[Official website](https://istio.io) ·
[License: Apache-2.0](https://github.com/istio/istio/blob/master/LICENSE)

## Overview

Istio is a service mesh for Kubernetes (and, secondarily, VMs). It intercepts service-to-service traffic and applies mutual TLS, traffic routing, retries, and telemetry without requiring changes to application code. It was started in 2017 by Google, IBM, and Lyft — Lyft contributing the Envoy proxy that Istio uses as its data plane[^1]. It became the de facto reference for what a mesh is, and much of the industry's vocabulary (sidecar, control plane, VirtualService) traces back to it. It joined the CNCF as an incubating project in 2022 and graduated in 2024[^2].

The defining tension of Istio is power versus operational cost. It exposes a large surface of CRDs and can express nearly any L7 routing, security, and observability policy — but historically it did this by injecting an Envoy sidecar into every pod, which adds memory, CPU, per-hop latency, and a second lifecycle to manage alongside the app. Much of Istio's recent history is a direct response to that complaint: consolidating a four-component control plane into a single binary, and building a sidecar-less "ambient" mode.

Istio is broad and moves quickly. It is not a set-and-forget dependency; running it in production means owning proxy upgrades, config debugging, and a short support window. The payoff is a uniform security and traffic layer that most teams could not build themselves.

## Getting Started

```bash
# Download istioctl and install a demo profile into the current kube-context
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y

# Enable automatic sidecar injection for a namespace, then (re)deploy
kubectl label namespace default istio-injection=enabled
kubectl rollout restart deployment -n default
```

```yaml
# Route 90/10 between two subsets — the canonical Istio pattern
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: ["reviews"]
  http:
    - route:
        - destination: { host: reviews, subset: v1 }
          weight: 90
        - destination: { host: reviews, subset: v2 }
          weight: 10
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
```

## Architecture / How It Works

Istio splits into a **control plane** and a **data plane**.

- **Istiod** is the control plane — a single Go binary since Istio 1.5 (2020), which merged the previously separate Pilot, Citadel, and Galley components into one process[^3]. It watches Kubernetes for Istio CRDs and Services, translates them into Envoy xDS configuration, distributes that config to every proxy, and acts as a CA issuing SPIFFE-based workload identities for mTLS.
- **The data plane** is where traffic actually flows. In the classic model, each pod gets an **Envoy** sidecar injected by a mutating admission webhook; all inbound/outbound traffic is redirected (via iptables or the CNI plugin) through that proxy.

Configuration is expressed through CRDs, the important ones being `VirtualService` (routing rules), `DestinationRule` (subsets, load-balancing, circuit breaking), `Gateway` (edge ingress/egress), `ServiceEntry` (external services), `PeerAuthentication` (mTLS mode), `AuthorizationPolicy` (L7 authz), and `Sidecar` (scoping which config a proxy receives). Istio also implements the upstream Kubernetes Gateway API, which is now the recommended way to configure ingress.

**Ambient mode** is the newer sidecar-less data plane, reaching GA in Istio 1.24 (2024)[^4]. It splits the proxy's job in two: a per-node **ztunnel** (a Rust proxy, in its own repo) handles L4 — mTLS, identity, and TCP routing — while optional per-namespace **waypoint** proxies (Envoy) handle L7 features like HTTP routing and authorization. The point is to pay the L7 cost only where you need it, and to avoid restarting every pod to upgrade the mesh.

The historical Mixer component — a separate telemetry/policy service each request called out to — was deprecated in 1.5 and removed; telemetry now runs as Envoy extensions in-proxy, eliminating a notorious latency and scaling bottleneck.

## Production Notes

- **Sidecar overhead is real.** Each Envoy adds baseline memory (tens of MB per pod) and per-hop latency (typically low single-digit milliseconds, but it compounds across call chains). At scale this is the single biggest driver toward ambient mode.
- **xDS push amplification.** Istiod pushes config to every connected proxy on relevant changes. In large meshes a churny cluster can generate expensive full pushes. The standard mitigation is the `Sidecar` resource to scope each proxy's config to only the services it talks to — without it, every proxy learns about every service.
- **Short support window.** Istio ships roughly quarterly and supports only about the latest three minor releases at once. Falling behind is a real operational risk; you will be upgrading a few times a year whether you want to or not.
- **Upgrades want canary, not in-place.** The safer path is revision-based canary upgrades (`istioctl install --revision`), running two control planes and migrating namespaces by label. In-place upgrades restart proxies fleet-wide and are harder to roll back.
- **mTLS PERMISSIVE is a footgun.** `PeerAuthentication` defaults to permissive (accept both plaintext and mTLS) so you don't break traffic during rollout. Teams forget to move to `STRICT` and believe they have zero-trust when they don't.
- **Config debugging is hard.** VirtualService/DestinationRule interactions, host matching, and gateway binding produce silent misroutes. `istioctl proxy-config` and `istioctl analyze` are essential; without them you are reading raw Envoy dumps.
- **Injection requires a restart.** Labeling a namespace does nothing to already-running pods — you must recreate them for sidecars to appear, a common first-day surprise.

## When to Use / When Not

**Use when:**
- You run many services on Kubernetes and need uniform mTLS, identity, and L7 policy without touching app code.
- You need fine-grained traffic control: weighted canaries, mirroring, fault injection, per-route retries and timeouts.
- You want rich, consistent telemetry (metrics, distributed tracing hooks, access logs) across polyglot services.

**Avoid when:**
- You have a handful of services — the operational cost outweighs the benefit; start with Kubernetes-native networking.
- You cannot commit to ongoing upgrades and an operator who understands Envoy; an unmaintained mesh is a liability.
- You only need edge ingress or an API gateway, not east-west mesh policy — a gateway alone is simpler.
- Latency budgets are extremely tight and you can't absorb per-hop proxy overhead (though ambient mode narrows this gap).

## Alternatives

- linkerd/linkerd2 — lighter, opinionated mesh with a purpose-built Rust micro-proxy; use it when you want simplicity and low overhead over Istio's feature breadth.
- cilium/cilium — eBPF-based networking with mesh features; use it when you want L3/L4 policy and observability in the kernel and less proxy footprint.
- hashicorp/consul — service mesh that spans Kubernetes, VMs, and multi-datacenter; use it when your workloads aren't all in one Kubernetes cluster.
- envoyproxy/gateway — Envoy-based gateway implementing the Kubernetes Gateway API; use it when you need ingress/API-gateway only, not east-west mesh.
- kong/kong — API gateway with plugins; use it when your problem is north-south API management rather than service-to-service policy.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2017-05 | First public release; Envoy sidecars, Pilot/Mixer/Citadel control plane[^1]. |
| 1.0 | 2018-07 | First stable release. |
| 1.5 | 2020-03 | Control plane consolidated into single istiod binary; Mixer deprecated[^3]. |
| 1.6 | 2020-05 | Telemetry v2 (in-proxy) default; smoother upgrade tooling. |
| 1.16 | 2022-11 | Ambient mesh introduced as experimental. |
| 1.24 | 2024-11 | Ambient mode reached GA[^4]. |

## References

[^1]: Istio launch announcement, "Introducing Istio" — 2017-05-24. https://istio.io/latest/news/releases/0.x/announcing-0.1/
[^2]: Cloud Native Computing Foundation — Istio project page and graduation. https://www.cncf.io/projects/istio/
[^3]: Istio blog, "Introducing istiod: simplifying the control plane" — 2020. https://istio.io/latest/blog/2020/istiod/
[^4]: Istio 1.24 release announcement (ambient mesh GA) — 2024-11. https://istio.io/latest/news/releases/1.24.x/announcing-1.24/
[^5]: Ambient mesh overview. https://istio.io/latest/docs/ambient/overview/

## Tags

service-mesh, kubernetes, envoy, go, mtls, traffic-management, observability, cloud-native, cncf, microservices, sidecar, ambient-mesh
