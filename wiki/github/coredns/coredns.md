# coredns/coredns

> A DNS server whose entire behavior is a chain of plugins configured by a Caddyfile-style Corefile — and the default cluster DNS of Kubernetes.

[GitHub repo](https://github.com/coredns/coredns) ·
[Official website](https://coredns.io) ·
[License: Apache-2.0](https://github.com/coredns/coredns/blob/master/LICENSE)

## Overview

CoreDNS is a DNS server written in Go by Miek Gieben, the author of the widely
used `miekg/dns` library[^1]. It is built on the Caddy server framework, and it
inherits Caddy's configuration model directly: the `Corefile` is a Caddyfile,
and each block wires up a stack of plugins that a query flows through in
order[^2]. There is almost no "core" DNS logic that is not itself a plugin —
even response caching, forwarding, and Kubernetes service discovery are plugins.
This is the defining design decision: CoreDNS trades raw feature breadth in the
binary for a small, composable core that you extend by recompiling with the
plugins you want.

Its center of gravity is Kubernetes. CoreDNS became the default cluster DNS in
Kubernetes 1.13 (2018), replacing kube-dns, and that is where the overwhelming
majority of deployments run today[^3]. It is a Cloud Native Computing Foundation
graduated project (graduated January 2019)[^4]. Outside Kubernetes it is used as
an authoritative server, a caching forwarder, and a service-discovery front end
over etcd or cloud provider APIs — but "the DNS server inside your cluster" is
what most operators mean when they say CoreDNS.

The tension worth naming up front: CoreDNS is flexible because plugins compose,
but that composition is compile-time and order-sensitive. The set of plugins in
a build is fixed by `plugin.cfg` at build time, and the order plugins run in is
defined by that file — not by the order you write them in the Corefile[^5]. Most
production surprises trace back to this.

## Getting Started

Pull the container (the normal path) or build from source with Go 1.25+:

```bash
docker run -d --name coredns -p 53:53/udp -p 53:53/tcp \
  -v "$PWD/Corefile:/Corefile" coredns/coredns -conf /Corefile
```

A minimal caching forwarder that logs queries and exposes metrics:

```corefile
.:53 {
    cache 30
    forward . 8.8.8.8 1.1.1.1 {
        policy sequential
    }
    prometheus :9153
    log
    errors
}
```

Test it:

```bash
dig @127.0.0.1 example.com
```

## Architecture / How It Works

A running CoreDNS server is a set of **server blocks** (the `zone:port { ... }`
stanzas in the Corefile). Each server block owns a **plugin chain**. A query
arriving for a zone is handed to the first plugin in that zone's chain; each
plugin either answers the query, mutates it, or calls the next plugin via
`plugin.Handler.ServeDNS`. This is ordinary middleware, expressed for DNS.

Two facts about the chain routinely trip people up:

1. **Chain order is fixed at build time, not by Corefile order.** The directive
   order lives in `plugin.cfg`, compiled into the binary by `coredns/caddy`'s
   directive registration[^5]. Writing `cache` after `forward` in your Corefile
   does not put cache after forward in the chain — the compiled order wins. To
   reorder, you rebuild with an edited `plugin.cfg`.
2. **The plugin set is fixed at build time.** Adding a plugin means editing
   `plugin.cfg` and recompiling (or setting `COREDNS_PLUGINS` during `make`).
   There is no runtime plugin loading. Out-of-tree plugins exist but require a
   custom build[^6].

The `kubernetes` plugin is the reason most people run CoreDNS. It opens a watch
against the Kubernetes API server for Services, Endpoints/EndpointSlices, and
Pods, holds that state in memory, and synthesizes A/AAAA/SRV/PTR records for the
`cluster.local` domain on the fly. It is a read-through into cluster state, not a
zone file. Because it holds cluster objects in memory and reacts to every change,
its resource use scales with the number of Services and Endpoints, not with query
volume.

Transport is handled by the server type registered for the block's scheme:
plain UDP/TCP (`dns://`), DoT (`tls://`), DoH over HTTP/2 (`https://`), DoH3
(`https3://`), DoQ (`quic://`), and gRPC (`grpc://`) all exist as distinct
server implementations sharing the same plugin chains[^7]. TLS termination is
CoreDNS's own unless you front it with a proxy.

## Production Notes

**Plugin ordering is the number-one footgun.** If `cache` appears to do nothing,
or `rewrite` runs at the wrong time, check the compiled chain order in
`plugin.cfg` rather than your Corefile. This is counterintuitive to anyone
expecting Corefile-order semantics.

**Kubernetes DNS latency is usually not CoreDNS.** The classic "DNS is slow in
our cluster" report is almost always the client-side `ndots:5` default in the
generated `/etc/resolv.conf`: a lookup for an external name like `api.github.com`
first tries five cluster-local search-domain permutations, producing four NXDOMAIN
round trips before the real query[^8]. Mitigations are client-side (`ndots:1`,
fully-qualified names with a trailing dot) or the `autopath` plugin, which
shortcuts the search-domain walk inside CoreDNS — at the cost of extra memory to
watch Pods.

**UDP conntrack races bite at scale.** On Linux with `iptables`/`conntrack`,
concurrent UDP DNS queries to a ClusterIP can hit a well-documented kernel race
that drops packets and surfaces as intermittent ~5s DNS timeouts (the resolver
retry interval)[^9]. The standard fix is **NodeLocal DNSCache** — a per-node
CoreDNS instance that serves over TCP upstream and keeps the hot path on the node
— not tuning CoreDNS itself.

**Sizing scales with the API, not queries.** Memory and CPU for the `kubernetes`
plugin track Service/Endpoint count and churn. Large or high-churn clusters need
CoreDNS memory limits raised and replica counts (or the cluster-proportional
autoscaler) set accordingly; the defaults shipped by kubeadm are conservative.

**`reload` is convenient and dangerous.** The `reload` plugin polls the Corefile
and applies changes without a restart, but a Corefile that fails to parse on
reload will keep the old config silently — validate changes out of band.

**Upgrades are gated by the deprecation policy.** CoreDNS announces a breaking
change one minor release ahead, ships it with lenient parsing in `x.(y+1).0`, and
removes the compatibility shim in `x.(y+1).1`[^10]. In Kubernetes, the CoreDNS
version is coupled to the cluster version, so cluster upgrades can quietly bump
CoreDNS across a breaking Corefile change — read the migration notes before
`kubeadm upgrade`.

## When to Use / When Not

**Use when:**
- You run Kubernetes — it is already the default; the question is how you tune it.
- You want a caching/forwarding resolver whose behavior is a short, auditable
  plugin chain rather than a monolithic config file.
- You need service discovery backed by etcd, or cloud provider records (route53,
  Azure, Google Cloud DNS) fronted by a uniform DNS interface.
- You want DoT/DoH/DoQ/gRPC transports from one server with metrics built in.

**Avoid when:**
- You need a full recursive resolver with DNSSEC validation — CoreDNS forwards and
  can serve signed zones, but Unbound or BIND are the mature recursive validators.
- You run large authoritative zones with heavy DNSSEC signing and zone-transfer
  workflows — the traditional authoritative servers (BIND, Knot, NSD) are built
  for that.
- You want to add functionality without recompiling — the build-time plugin model
  is a poor fit for teams that cannot own a custom binary.

## Alternatives

- powerdns/pdns — use instead when you want a database-backed authoritative
  server with a rich API and dynamic backends outside a cluster.
- NLnetLabs/unbound — use instead when you need a hardened, validating recursive
  resolver rather than a forwarder.
- isc-projects/bind9 — use instead for large authoritative DNSSEC deployments and
  standards-completeness.
- NLnetLabs/knot / CZ-NIC Knot DNS — use instead for high-throughput
  authoritative serving with online signing.
- kubernetes/dns (kube-dns) — the predecessor CoreDNS replaced; only relevant on
  very old clusters.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2016-03 | Forked config and server model from the Caddy framework[^2]. |
| CNCF sandbox | 2017 | Accepted as a CNCF-hosted project. |
| k8s default | 2018 | Became the default cluster DNS in Kubernetes 1.13, replacing kube-dns[^3]. |
| CNCF graduated | 2019-01 | Graduated within the CNCF[^4]. |
| DoH / DoQ / DoH3 | 2020–2023 | HTTPS, QUIC, and HTTP/3 transports added alongside UDP/TCP/TLS/gRPC[^7]. |

## References

[^1]: `miekg/dns` — the Go DNS library CoreDNS is built on. https://github.com/miekg/dns
[^2]: CoreDNS is built on the Caddy server framework; the Corefile is a Caddyfile. https://coredns.io/manual/toc/ and https://github.com/coredns/caddy
[^3]: Kubernetes blog, "CoreDNS GA for Kubernetes Cluster DNS." https://kubernetes.io/blog/2018/07/10/coredns-ga-for-kubernetes-cluster-dns/
[^4]: CNCF, "CoreDNS graduation." https://www.cncf.io/announcements/2019/01/24/coredns-graduation/
[^5]: Plugin chain order is defined by `plugin.cfg`, not Corefile order. https://github.com/coredns/coredns/blob/master/plugin.cfg
[^6]: External plugins require a custom build. https://coredns.io/explugins/
[^7]: Supported transports (UDP/TCP, DoT, DoH, DoH3, DoQ, gRPC), per README. https://github.com/coredns/coredns
[^8]: Kubernetes DNS `ndots:5` and search-domain behavior. https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
[^9]: "Racy conntrack and DNS lookup timeouts" — the 5s UDP DNS timeout on Linux/iptables. https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts
[^10]: CoreDNS deprecation policy, per README. https://github.com/coredns/coredns#deprecation-policy

## Tags

dns, dns-server, go, kubernetes, cncf, service-discovery, plugin-architecture, caddy, networking, self-hosted
