# Netflix/ribbon

> Client-side IPC and software load-balancing library for the JVM — the load balancer inside classic Spring Cloud Netflix, now in maintenance mode.

[GitHub repo](https://github.com/Netflix/ribbon) ·
[License: Apache-2.0](https://github.com/Netflix/ribbon/blob/master/LICENSE)

## Overview

Ribbon is a Java inter-process-communication library built at Netflix around 2013 to give client applications load balancing, retries, and dynamic server-list discovery without a dedicated proxy tier[^1]. Instead of routing REST calls through a middlebox, each client holds a load balancer in-process, picks a server from a list (often supplied by Eureka), and applies a pluggable rule (round-robin, availability-filtering, zone-affinity). This "client-side load balancing" model was central to the first generation of Netflix's cloud microservice stack, alongside Eureka (discovery), Hystrix (circuit breaking), and Zuul (edge gateway).

For most JVM developers Ribbon arrived indirectly, through Spring Cloud Netflix, where it was the default load balancer under `@LoadBalanced RestTemplate` and Feign for years. That is also the source of its defining tension: the project has been in **maintenance mode since ~2016**, Netflix having moved internal RPC to a gRPC-based stack with load-balancing and discovery interceptors[^2], while Spring Cloud deprecated and then removed Ribbon in favour of Spring Cloud LoadBalancer[^3]. The code is still deployed at scale inside Netflix for some modules, but no new features land, and the reactive `ribbon`, `ribbon-transport`, and RxNetty-based pieces are explicitly marked "not used" by the maintainers[^1].

Read Ribbon today as a stable-but-frozen dependency you likely inherit through an older Spring Cloud release, not as something to adopt fresh. Recent commit activity is dependency and build housekeeping, not development.

## Getting Started

```xml
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon-loadbalancer</artifactId>
    <version>2.7.18</version>
</dependency>
```

```java
// Fixed server list + availability-filtering rule, no discovery
List<Server> servers = Arrays.asList(
        new Server("localhost", 8080),
        new Server("localhost", 8088));

BaseLoadBalancer lb = LoadBalancerBuilder.newBuilder()
        .withRule(new AvailabilityFilteringRule())
        .buildFixedServerListLoadBalancer(servers);

Server chosen = lb.chooseServer(null);   // apply the rule
// ... make your HTTP/TCP call against chosen.getHost():chosen.getPort()
lb.markServerDown(chosen);               // feed health back to the balancer
```

Verify the exact current version against Maven Central[^4] — the `2.2.2` shown in the project README predates several patch releases.

## Architecture / How It Works

Ribbon is a set of loosely related modules rather than one library, and the module you pick matters more than the version:

- **ribbon-core** — client configuration (`IClientConfig`) and shared abstractions. Deployed at scale internally.
- **ribbon-loadbalancer** — the durable core: `ILoadBalancer`, `Server`, `IRule`, `IPing`, `ServerList`, `ServerListFilter`. Usable standalone with a static server list. This is the part most people actually want.
- **ribbon-eureka** — supplies a dynamic `ServerList` from a Eureka client so the balancer tracks registrations in real time.
- **ribbon-httpclient** — a REST client on Apache HttpClient wired to the load balancer. Deprecated; the README notes Netflix uses everything except its SSL package and layers an internal client on top.
- **ribbon-transport** — RxNetty-based HTTP/TCP/UDP reactive transport. Marked not used.
- **ribbon** — the top-level reactive API (`Ribbon.createHttpResourceGroup`, annotation proxies) integrating with Hystrix. Also marked not used.

The load-balancer core follows a strategy pattern. An `IRule` decides which server to return: `RoundRobinRule`, `WeightedResponseTimeRule`, `AvailabilityFilteringRule` (skips servers tripping circuits or over a connection limit), and `ZoneAwareLoadBalancer` (partitions servers by AWS availability zone and prefers the local zone). An `IPing` plus `ServerListUpdater` refresh liveness, and a `ServerListFilter` can prune the candidate set (for example zone-affinity). Configuration is per-named-client and, in the Spring Cloud world, lives under `<clientName>.ribbon.*` properties.

The reactive layers (`ribbon`, `ribbon-transport`) are built on RxJava 1 and RxNetty, both effectively legacy. This is why those modules aged worst: the surrounding reactive ecosystem moved to Reactor and RxJava 2/3, and Netflix stopped investing before the migration.

## Production Notes

- **You probably didn't choose Ribbon; Spring Cloud did.** Its real-world blast radius comes from `@LoadBalanced` `RestTemplate` and Feign clients in older Spring Cloud Netflix apps. If you are on Spring Cloud 2020.0 (Ilford) or later, Ribbon is gone and the equivalent is Spring Cloud LoadBalancer — a migration, not a drop-in swap[^3].
- **Eager vs lazy client init.** A classic footgun: Ribbon clients initialize lazily on first request, so the first call to each downstream can pay list-fetch and ping latency, occasionally surfacing as timeouts under cold start. Teams commonly force eager loading via `ribbon.eager-load.enabled=true` and an explicit client list in Spring Cloud.
- **Read timeout semantics with retries.** Ribbon's `MaxAutoRetries` / `MaxAutoRetriesNextServer` interact with `ReadTimeout` and `ConnectTimeout` per client. Misconfigured, a "retry next server" on a non-idempotent POST can double-execute side effects. Confirm idempotency before enabling next-server retries.
- **Stale server lists.** Discovery-backed lists are refreshed on a timer (`ServerListRefreshInterval`), so a just-terminated instance can still be chosen until the next refresh; `AvailabilityFilteringRule` and ping mitigate but do not eliminate this.
- **Frozen means frozen.** Bugs that are not security-critical are unlikely to be fixed upstream; the maintainers accept complete external PRs but do not prioritize feature requests[^1]. Treat it as you would any library on life support: pin the version, avoid the RxNetty modules, and plan an exit.
- **No official homepage or current docs site.** The README, the `ribbon-examples` module, and the Spring Cloud Netflix reference (for the deprecated integration) are the practical documentation.

## When to Use / When Not

**Use when:**
- You are maintaining an existing Spring Cloud Netflix / Ribbon deployment and need to understand or tune it rather than rewrite it.
- You want a small, standalone JVM client-side load balancer over a static or Eureka-fed server list and can live with a frozen dependency (`ribbon-loadbalancer` only).

**Avoid when:**
- You are starting a new service today — pick a maintained load balancer instead.
- You want reactive HTTP transport — the RxNetty modules are unmaintained and pin legacy RxJava 1.
- You need vendor support, security patches, or new protocol features.

## Alternatives

- spring-cloud/spring-cloud-loadbalancer — the direct successor for Spring Cloud users; use this when you are on Spring Cloud 2020.0+ and Ribbon has been removed.
- grpc/grpc-java — use when you want Netflix's own chosen direction: gRPC with client-side load-balancing and multi-language support.
- envoyproxy/envoy — use when you'd rather move load balancing out of the JVM process into a sidecar/service mesh.
- resilience4j/resilience4j — use when your real need was the fault-tolerance/circuit-breaking half (the Hystrix successor), composed with a modern HTTP client.
- Netflix/eureka — use alongside any of the above when you still need service discovery; Ribbon's discovery integration was always Eureka-based.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-01 | Open-sourced from Netflix's cloud platform as client-side IPC + load balancing[^1]. |
| ~2.x | 2015 | Reactive `ribbon`/`ribbon-transport` APIs on RxJava 1 + RxNetty added. |
| maintenance | ~2016 | Placed in maintenance mode; Netflix pivots RPC to gRPC-based stack[^2]. |
| deprecation | ~2019–2020 | Spring Cloud deprecates Ribbon, ships Spring Cloud LoadBalancer as replacement[^3]. |
| 2.7.x | ongoing | Patch/dependency releases only; no feature development. |

(Exact tag dates should be confirmed against the GitHub releases page[^5]; only the coarse timeline is asserted here.)

## References

[^1]: Netflix/ribbon README, "Project Status: On Maintenance" and module list. https://github.com/Netflix/ribbon
[^2]: Netflix Technology Blog on the move from Ribbon/Eureka to a gRPC-based RPC stack. https://netflixtechblog.com/
[^3]: Spring Cloud, "Spring Cloud Netflix" — Ribbon deprecation and Spring Cloud LoadBalancer replacement. https://spring.io/projects/spring-cloud-netflix
[^4]: Ribbon artifacts on Maven Central. https://search.maven.org/search?q=g:com.netflix.ribbon
[^5]: Ribbon releases. https://github.com/Netflix/ribbon/releases

## Tags

java, jvm, load-balancing, client-side-load-balancing, microservices, service-discovery, netflix-oss, spring-cloud, rpc, ipc, maintenance-mode
