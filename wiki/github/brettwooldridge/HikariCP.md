# brettwooldridge/HikariCP

> A small, fast JDBC connection pool for the JVM — the default pool in Spring Boot since 2.0.

[GitHub repo](https://github.com/brettwooldridge/HikariCP) ·
[Maven Central](https://central.sonatype.com/artifact/com.zaxxer/HikariCP) ·
[License: Apache-2.0](https://github.com/brettwooldridge/HikariCP/blob/dev/LICENSE)

## Overview

HikariCP is a JDBC connection pool written by Brett Wooldridge, first released in 2013[^1]. A connection pool keeps a set of already-open database connections alive and hands them out on demand, so application code pays the cost of the TCP + TLS + auth handshake once at startup rather than on every query. HikariCP's pitch is that it does this with the least overhead of any Java pool — the library is roughly 165 KB and the hot path (`getConnection()` / `close()`) is engineered to add close to zero cost on top of the raw driver[^2].

Its defining characteristic is aggressive micro-optimization in service of a very small feature set. Where older pools (DBCP, c3p0) accumulated configuration surface and validation machinery, HikariCP deliberately ships fewer knobs, sane defaults, and a hand-written concurrency core. The tradeoff is philosophical: HikariCP will refuse to add features it considers footguns (notably XA/distributed transactions, which require an external transaction manager), and its maintainer is opinionated about "correct" usage — most visibly around pool sizing, where the project argues that small fixed-size pools outperform large ones by a wide margin[^3].

The reason HikariCP matters far beyond its own star count is distribution: Spring Boot switched its default `DataSource` implementation from the Tomcat JDBC pool to HikariCP in Spring Boot 2.0 (2018)[^4]. For a large fraction of Java web services, HikariCP is therefore the pool in production whether or not the team chose it deliberately.

## Getting Started

```xml
<!-- Maven, Java 11+ -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>7.1.0</version>
</dependency>
```

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/app");
config.setUsername("app");
config.setPassword("secret");
config.setMaximumPoolSize(10);          // fixed-size pool (recommended)
config.setConnectionTimeout(30_000);    // ms to wait for a free connection
config.setMaxLifetime(1_800_000);       // retire connections after 30 min

try (HikariDataSource ds = new HikariDataSource(config);
     Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement("SELECT 1")) {
    ps.execute();
}   // conn.close() returns the connection to the pool, it is not physically closed
```

On Spring Boot, no code is needed — set `spring.datasource.hikari.*` properties and the auto-configuration wires it up.

## Architecture / How It Works

HikariCP's speed comes from a handful of custom internals rather than one big trick:

- **`ConcurrentBag`** — a purpose-built lock-free-ish collection that stores pooled connections. It uses thread-local lists so a thread returning a connection and then re-borrowing tends to get the same one back without contending on shared state, falling back to a shared queue (with `SynchronousQueue`-style handoff) under contention. This is the core reason borrow/return is cheap under load.
- **`FastList`** — an `ArrayList` replacement that drops the bounds-checking and `modCount` bookkeeping HikariCP doesn't need, used to track open `Statement` objects per connection so they can be closed automatically.
- **Proxy layer** — the `Connection`, `Statement`, and `ResultSet` objects handed to callers are thin proxies (`ProxyConnection` etc.) that intercept `close()` to recycle rather than destroy, and route everything else to the real driver object. Historically HikariCP generated these with Javassist bytecode manipulation; the modern codebase uses hand-written delegates.
- **`HouseKeeper`** — a scheduled task that enforces `idleTimeout`, `maxLifetime`, and `keepaliveTime`, retiring and replacing connections in the background so borrow-time stays fast.

The design leans hard on the JIT: methods are kept small and monomorphic so they inline. This is why HikariCP is sensitive to things most libraries ignore — for example it depends on an accurate system clock, and the project explicitly warns that a VM with unsynchronized time (NTP not configured inside the guest) will produce spurious timeouts and misbehavior[^2].

Notably absent: XA/two-phase-commit support. HikariCP does not pool `XADataSource`; distributed transactions require a real transaction manager (Bitronix, Narayana, Atomikos) wrapping the pool.

## Production Notes

- **Pool sizing is counterintuitive.** The maintainer's most-cited guidance is that a *small* pool is faster than a large one, because a database with N cores can only truly execute a bounded number of queries in parallel; oversized pools add context-switching and lock contention. A common starting formula is `connections = (core_count * 2) + effective_spindle_count`. Teams routinely set `maximumPoolSize` to 100+ "to be safe" and make throughput worse[^3].
- **Set `maxLifetime` shorter than any upstream connection limit.** Databases, load balancers, and cloud proxies (e.g. RDS Proxy, HAProxy, firewalls) silently drop idle TCP connections. If HikariCP believes a connection is alive but the network has killed it, the first query fails. `maxLifetime` should be several seconds below the shortest such limit (default is 30 min).
- **Configure TCP keepalive.** The project flags a rare failure mode where a pool can drain to zero and fail to recover; the mitigation is OS-level or driver-level TCP keepalive (e.g. `tcpKeepAlive=true` on PostgreSQL)[^5]. This is called out prominently in the README as an important operational step.
- **`connectionTimeout` failures are usually not HikariCP's fault.** A `SQLTransientConnectionException` after 30 s almost always means the pool is exhausted — connections are being borrowed and not returned (leaked) or held too long by slow queries. Enable `leakDetectionThreshold` (e.g. 60_000 ms) to log stack traces of connections held longer than expected.
- **`minimumIdle` defaults to `maximumPoolSize`** — i.e. a fixed-size pool. The project recommends leaving it that way; setting a lower `minimumIdle` trades steady-state responsiveness for the appearance of a smaller footprint and is rarely worth it.
- **Metrics.** First-class integration with Dropwizard Metrics and Micrometer exposes pool usage, wait time, and connection-creation time — the single most useful signal for diagnosing exhaustion. Instrument it before you need it.

## When to Use / When Not

**Use when:**
- You need a JDBC connection pool for any modern JVM service (Java 11+). It is the sensible default.
- You're on Spring Boot and want to stay on the supported happy path.
- You care about tail latency and want the pool to be the part of the stack you don't have to think about.

**Avoid / look elsewhere when:**
- You need XA/distributed transactions managed by the pool itself — pair a transaction manager with an XA-aware pool (Agroal, or a JTA container) instead.
- You're stuck on Java 8 or older — you can still use it, but you're pinned to the frozen 4.0.3 (Java 8) / 2.4.13 (Java 7) artifacts, which no longer receive fixes.
- You're on Quarkus, where Agroal is the integrated default and better wired into the framework's transaction and health machinery.

## Alternatives

- agroal/agroal — use when you're on Quarkus or need pool-managed XA with Narayana; it's built for JTA integration.
- apache/tomcat (tomcat-jdbc pool) — use when you're already inside Tomcat/older Spring and want the pre-2.0 default; more knobs, heavier.
- apache/commons-dbcp — use when a legacy app already depends on it; mature but slower and more configuration surface.
- vibur/vibur-dbcp — use when you want a concurrent pool with built-in statement caching and are comparing benchmarks.
- swaldman/c3p0 — legacy; avoid for new projects, relevant mainly when maintaining old code that already uses it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2013 | Initial release; Javassist-based proxy generation, ConcurrentBag[^1]. |
| 2.x | 2015–2017 | Java 6/7/8 artifact split (`HikariCP-java6/7`); FastList, spike-demand tuning. |
| — | 2018-03 | Adopted as the default pool in Spring Boot 2.0[^4]. |
| 3.x | 2018 | Java 8 baseline; dropped older Java artifacts from the main line. |
| 4.0.3 | 2021 | Last release supporting Java 8. |
| 5.0 | 2021 | Java 11 baseline; Java 8 support frozen at 4.0.3. |
| 6.x | 2024 | Continued Java 11+ line. |
| 7.1.0 | 2026 | Current release, Java 11+ artifact `com.zaxxer:HikariCP`. |

## References

[^1]: HikariCP repository, brettwooldridge/HikariCP (created 2013). https://github.com/brettwooldridge/HikariCP
[^2]: HikariCP wiki, "Down the Rabbit Hole" — internals and micro-optimizations. https://github.com/brettwooldridge/HikariCP/wiki/Down-the-Rabbit-Hole
[^3]: HikariCP wiki, "About Pool Sizing." https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
[^4]: Spring Boot 2.0 Release Notes — HikariCP as the default DataSource. https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.0-Release-Notes
[^5]: HikariCP README / wiki, "Setting Driver or OS TCP Keepalive." https://github.com/brettwooldridge/HikariCP/wiki/Setting-Driver-or-OS-TCP-Keepalive

## Tags

java, jdbc, connection-pool, database, high-performance, spring-boot, jvm, sql, infrastructure
