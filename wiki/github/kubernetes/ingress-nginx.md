# kubernetes/ingress-nginx

> The long-dominant NGINX-based Ingress controller for Kubernetes — now retired, with no further releases or security fixes after March 2026.

[GitHub repo](https://github.com/kubernetes/ingress-nginx) ·
[Official website](https://kubernetes.github.io/ingress-nginx/) ·
[License: Apache-2.0](https://github.com/kubernetes/ingress-nginx/blob/main/LICENSE)

## Overview

ingress-nginx is an Ingress controller: a Kubernetes workload that watches
`Ingress` (and, in later versions, `Service`/`EndpointSlice`) objects and
renders them into a running NGINX reverse proxy. For most of Kubernetes'
history it was the default answer to "how do I expose HTTP services in my
cluster" — bundled or one-command-installable on nearly every distribution,
and the controller most tutorials assumed. As of 2026 it is the archived,
retired project it announced it would become[^1].

Two things dominate any honest read of this repo. First, it is **retired**.
The Kubernetes project announced in November 2025 that best-effort
maintenance would end in March 2026, after which there are no releases, no
bugfixes, and no security patches[^1]. The GitHub repository is archived
(read-only); existing Helm charts and container images remain hosted, but the
code is frozen. Second, its final years were shaped by a severe security
event — the March 2025 "IngressNightmare" cluster of vulnerabilities
(CVE-2025-1974 and related) that allowed unauthenticated remote code
execution via the admission controller[^2]. That class of exposure, rooted in
the controller's design of turning user-supplied annotations into NGINX
config, is a large part of why the maintainers steered users toward Gateway
API implementations instead of investing further here.

Do not confuse this project with `nginxinc/kubernetes-ingress`, a separate,
still-maintained controller developed by F5/NGINX Inc. with overlapping
purpose, different annotations, and different CRDs. The two are frequently
mixed up because both are "NGINX Ingress controllers."

## Getting Started

New adopters should not deploy this — the README itself says so and points to
Gateway API[^3]. For reference, the historical install was a single Helm
release:

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

A minimal `Ingress` routed by the controller:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

## Architecture / How It Works

The controller runs as a Deployment (or DaemonSet) of pods, each containing
the Go control process and an NGINX (built on OpenResty) worker set in the
same container. The Go process is a Kubernetes informer client: it watches
`Ingress`, `Service`, `EndpointSlice`, `Secret`, and `ConfigMap` objects,
builds an in-memory model of the desired routing, and templates a full
`nginx.conf` from it.

The critical performance decision is how backend changes are applied. Early
versions wrote a new `nginx.conf` and triggered an NGINX reload on every
endpoint change, which caused connection drops and CPU spikes in churny
clusters. Later versions moved endpoint/upstream data into a **Lua** layer
(`lua-nginx-module`): pod IP changes are pushed to NGINX workers over a
shared dictionary without a config reload. Reloads still happen for
structural changes (new hosts, TLS, annotations, most ConfigMap edits) but
not for routine scaling.

Configuration has three surfaces, in increasing danger:

- **Per-Ingress annotations** (`nginx.ingress.kubernetes.io/*`) — the primary
  API; each maps to a piece of generated NGINX config.
- **Global ConfigMap** — cluster-wide NGINX tuning (timeouts, buffer sizes,
  `worker-processes`, etc.).
- **Snippet annotations** (`configuration-snippet`, `server-snippet`) — raw
  NGINX directives injected verbatim into the generated config. This is the
  feature that made arbitrary-config-injection attacks possible; snippets are
  disabled by default in recent versions and must be explicitly enabled[^2].

A **validating admission webhook** checks new Ingress objects by rendering
and test-loading the resulting NGINX config before admission. This webhook —
network-reachable inside the cluster and running config generation on
attacker-influenced input — was the vector for IngressNightmare[^2].

## Production Notes

- **Security posture is now static.** After March 2026 no CVE will be patched
  upstream[^1]. Any continued production use means either accepting frozen,
  potentially-vulnerable code or maintaining a private fork. Treat this as the
  single most important operational fact.
- **The admission webhook is a real attack surface.** Lock it down: network
  policy restricting who can reach it, keep snippet annotations disabled, and
  in multi-tenant clusters assume anyone who can create an Ingress can
  influence NGINX config. The README explicitly warns against multi-tenant
  use because Ingress creators are effectively cluster operators[^3].
- **Reloads still bite.** Structural changes (TLS cert rotation, adding hosts,
  annotation edits) trigger NGINX reloads; under heavy config churn or very
  large numbers of Ingresses, reload time and memory grow. The Lua endpoint
  path helps only with pod scaling, not config structure.
- **Annotation semantics are controller-specific.** `rewrite-target`, canary
  routing, and auth annotations do not transfer to any other controller.
  Migrations are rewrites, not config ports.
- **`ingressClassName` vs the old annotation.** Older clusters used
  `kubernetes.io/ingress.class`; current setups use the `ingressClassName`
  field and an `IngressClass` object. Mixing them silently drops routes.
- **The exit path is Gateway API.** The maintainers' recommended successor is
  a Gateway API implementation, not another Ingress controller[^1][^3].
  Gateway API expresses routing as `Gateway`/`HTTPRoute` CRDs with a
  role-oriented model that avoids the annotation-injection design.

## When to Use / When Not

**Use when:**
- You already run it in production and need a documented, stable target while
  you plan a migration — the frozen artifacts still work[^1].
- You are reading existing clusters/manifests and need to understand the
  annotation and Lua-reload behavior that shaped them.

**Avoid when:**
- You are starting fresh — pick a maintained Gateway API implementation
  instead[^3].
- You need ongoing security patching or vendor support of any kind.
- You run multi-tenant clusters where untrusted users can create Ingress
  objects[^3].

## Alternatives

- kubernetes-sigs/gateway-api — the successor API the maintainers point to; not a controller itself, but the standard its implementations follow.
- envoyproxy/gateway — Envoy-based Gateway API implementation; strong choice for greenfield L7 routing.
- nginxinc/kubernetes-ingress — F5/NGINX Inc.'s separate, still-maintained NGINX controller; closest drop-in if you specifically need NGINX.
- traefik/traefik — Go-native ingress/gateway with its own CRDs and dynamic config; use when you want built-in dashboard and Let's Encrypt.
- projectcontour/contour — Envoy-based, uses `HTTPProxy` CRD and Gateway API; use for Envoy features without hand-writing Envoy config.
- kong/kubernetes-ingress-controller — use when you want an API gateway (auth, rate limiting, plugins) fused with ingress.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-11 | Repository created; NGINX controller splits from the earlier combined ingress repo.[^4] |
| v1.0.0 | 2021-08 | Graduates to 1.x; requires the stable `networking.k8s.io/v1` Ingress API. |
| v1.9.x | 2023 | Hardening of annotation validation; snippet restrictions tightened. |
| — | 2025-03 | "IngressNightmare" (CVE-2025-1974 and related) disclosed — unauthenticated RCE via the admission controller.[^2] |
| — | 2025-11-11 | Retirement announced: maintenance ends March 2026.[^1] |
| v1.15.1 | 2026 (final line) | Last supported minor line; NGINX 1.27.1, k8s 1.31–1.35.[^5] |
| — | 2026-03 | Best-effort maintenance ends; repository archived.[^1] |

## References

[^1]: Kubernetes Blog, "What You Need to Know about Ingress NGINX Retirement" — 2025-11-11. https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/
[^2]: Wiz Research, "IngressNightmare: CVE-2025-1974 and related RCE vulnerabilities in ingress-nginx" — 2025-03. https://www.wiz.io/blog/ingress-nginx-kubernetes-vulnerabilities and https://nvd.nist.gov/vuln/detail/CVE-2025-1974
[^3]: ingress-nginx README, "Usage warnings" and Gateway API guidance. https://github.com/kubernetes/ingress-nginx#usage-warnings
[^4]: GitHub repository metadata, `kubernetes/ingress-nginx` (created 2016-11-04). https://github.com/kubernetes/ingress-nginx
[^5]: ingress-nginx README, "Supported Versions table." https://github.com/kubernetes/ingress-nginx#supported-versions-table

## Tags

go, kubernetes, ingress-controller, nginx, networking, reverse-proxy, load-balancer, cloud-native, retired, gateway-api
