# knative/serving

> Kubernetes-based scale-to-zero serverless containers: run any HTTP container, pay for it only while it serves requests.

[GitHub repo](https://github.com/knative/serving) ·
[Official website](https://knative.dev/docs/serving/) ·
[License: Apache-2.0](https://github.com/knative/serving/blob/main/LICENSE)

## Overview

Knative Serving is the request-serving half of Knative, a project started at Google in 2018[^1] to bring serverless workload semantics — scale-to-zero, request-driven autoscaling, revision snapshots, traffic splitting — to plain Kubernetes without locking you to a cloud FaaS. You give it an ordinary OCI container that listens on a port; it gives you a URL, autoscales the container between zero and N replicas based on in-flight request concurrency, and keeps every deploy as an immutable Revision you can route traffic to by percentage. Google Cloud Run is the best-known managed implementation of this same API surface.

The defining tradeoff is that Knative is infrastructure, not a product. It is a control plane and data plane that you install into a cluster you already run, on top of a networking layer you have to choose and operate (Istio, Contour, Kourier, or Gateway API). It is powerful precisely because it is generic — any container, any language — but that generality means there is no "function" abstraction, no build pipeline, and no batteries-included dashboard. The other half of the project, knative/eventing, handles event delivery; Serving only handles synchronous HTTP request serving.

Knative was accepted as a CNCF incubating project in March 2022[^2], and Serving reached 1.0 in November 2021[^3] after roughly three and a half years of pre-1.0 churn that broke APIs frequently. Since 1.0 it has settled into a predictable minor-release cadence and is actively maintained.

## Getting Started

Knative Serving requires an existing Kubernetes cluster plus a networking layer. The `quickstart` plugin sets up a local kind/minikube cluster with Kourier for evaluation:

```bash
# install the kn CLI + quickstart plugin, then:
kn quickstart kind        # creates a local cluster with Serving + Kourier + DNS
```

A Knative Service is a single YAML that collapses Route + Configuration + Revision:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: ghcr.io/knative/helloworld-go:latest
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "World"
```

```bash
kubectl apply -f hello.yaml
kn service list          # shows the URL; first request cold-starts a pod from zero
```

## Architecture / How It Works

Serving defines four CRDs that layer on each other:

- **Revision** — an immutable snapshot of a container image plus config. Every change mints a new Revision; you never mutate one.
- **Configuration** — the desired state that produces Revisions.
- **Route** — maps a URL and traffic percentages onto one or more Revisions (this is how blue/green and canary work).
- **Service** — the top-level convenience object that owns a Configuration and a Route together.

The data plane is where the serverless behavior lives. Every Revision pod runs a **queue-proxy** sidecar in front of the user container. queue-proxy enforces the per-pod concurrency limit, buffers requests, and reports concurrency metrics to the **autoscaler**. The default autoscaler is the **KPA** (Knative Pod Autoscaler), which scales on request concurrency (default target 100 in-flight per pod) rather than CPU, and — unlike Kubernetes' built-in HPA — can scale all the way to zero. You can opt a Revision back onto the HPA if you want CPU/memory-based scaling, at the cost of losing scale-to-zero.

The piece that makes scale-to-zero work is the **activator**. When a Revision is scaled to zero, ingress routes requests to the activator instead of to (nonexistent) pods. The activator holds the request open, signals the autoscaler that demand exists, waits for the first pod to become ready, then proxies the buffered request through. Once pods exist and there is enough capacity, the activator takes itself out of the request path. This buffering hop is the source of both the scale-to-zero magic and the cold-start latency.

Ingress is pluggable through a `KIngress` abstraction implemented by `net-*` adapters: **net-istio**, **net-contour**, **net-kourier**, and the newer **net-gateway-api**. Early Knative effectively required Istio; Kourier (a lightweight Envoy-based option) later became the common default for installs that don't otherwise need a service mesh.

## Production Notes

- **Cold starts are real and unavoidable at zero.** The first request after scale-to-zero pays for image pull (if not cached on the node), pod scheduling, container start, and readiness. For latency-sensitive services, set `autoscaling.knative.dev/min-scale: "1"` to keep a warm pod — which means you are no longer scale-to-zero and are paying for that replica continuously.
- **The activator can become a bottleneck.** Under scale-from-zero bursts, all traffic funnels through the activator until pods absorb it. The `target-burst-capacity` setting controls when the activator stays in-path; tuning it wrong either hurts burst latency or adds a permanent proxy hop.
- **Concurrency, not RPS, is the default scaling signal.** If your handler is slow, concurrency accumulates and you scale aggressively; if it's fast, you may under-scale relative to RPS expectations. Understand `containerConcurrency` and the autoscaler target before load-testing.
- **You must operate a networking layer.** Istio brings a full mesh and its own upgrade/operational burden; Kourier is lighter but has fewer features. Version skew between Serving and the `net-*` adapter is a recurring upgrade footgun — they release in lockstep and mismatches cause silent routing failures.
- **DNS and TLS are your problem.** Dev installs use magic DNS (sslip.io); production needs real wildcard DNS pointed at the ingress plus cert-manager or provided certificates. This is not automatic.
- **Revision garbage collection** is on by default but worth verifying — long-lived Configurations accumulate stale Revisions that clutter the cluster if GC is misconfigured.
- **Upgrades are cluster-admin, CRD-level operations.** Knative installs cluster-scoped CRDs and webhooks; upgrading is not a rolling app deploy, and skipping minor versions is not supported — upgrade one minor at a time.

## When to Use / When Not

**Use when:**
- You run your own Kubernetes and want Cloud Run-style scale-to-zero without leaving it.
- Traffic is spiky or intermittent and paying for idle replicas is wasteful.
- You want revision-based canary/blue-green traffic splitting as a first-class primitive.
- You want a vendor-neutral serverless API you could later move to a managed Knative provider.

**Avoid when:**
- You don't already operate Kubernetes — the install and networking-layer burden dwarfs the benefit.
- Your services are always-on and steady-traffic; a plain Deployment + HPA is simpler and has no activator hop.
- You need a true functions/FaaS developer experience with builds and a source-to-URL pipeline out of the box — Knative gives you containers, not functions.
- Cold-start latency is unacceptable and you can't afford a warm `min-scale` replica.

## Alternatives

- kedacore/keda — event-driven autoscaling (including scale-to-zero via an HTTP add-on) that bolts onto existing Deployments; use when you want autoscaling without adopting a full serverless platform and its CRDs.
- openfaas/faas — opinionated functions-as-a-service with a build pipeline and templates; use when you want a real function DX rather than raw containers.
- fission/fission — Kubernetes-native FaaS optimized for low cold-start via pre-warmed pools; use when function cold-start latency is the priority.
- kubernetes/kubernetes — the built-in HorizontalPodAutoscaler covers CPU/memory scaling with no extra install; use when you don't need scale-to-zero or request-concurrency scaling.
- knative/eventing — not an alternative but the complementary Knative component for asynchronous event delivery; pair it with Serving for event-driven workloads.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018-07 | First public release; announced alongside the Knative launch by Google[^1]. |
| 1.0 | 2021-11 | API stability milestone after ~3.5 years of pre-1.0 breaking changes[^3]. |
| CNCF incubation | 2022-03 | Knative accepted as a CNCF incubating project[^2]. |
| 1.x | 2021–2026 | Steady minor-release cadence; Gateway API networking support matured (`net-gateway-api`). |

## References

[^1]: Google Open Source Blog, "Knative: bringing serverless to Kubernetes everywhere" — 2018-07-24. https://cloud.google.com/blog/products/gcp/knative
[^2]: CNCF, "Knative accepted as a CNCF incubating project" — 2022-03-02. https://www.cncf.io/blog/2022/03/02/knative-accepted-as-a-cncf-incubating-project/
[^3]: Knative blog, "Knative 1.0" release announcement — 2021-11. https://knative.dev/blog/articles/knative-uncommon-1-0/

## Tags

serverless, kubernetes, autoscaler, scale-to-zero, go, cloud-native, containers, cncf, request-driven, knative
