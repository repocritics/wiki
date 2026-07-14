# apache/tomcat

> The reference servlet container — the Java web server that runs Servlet, JSP, EL, and WebSocket apps, and the default HTTP layer inside Spring Boot.

[GitHub repo](https://github.com/apache/tomcat) ·
[Official website](https://tomcat.apache.org) ·
[License: Apache-2.0](https://github.com/apache/tomcat/blob/main/LICENSE)

## Overview

Apache Tomcat is an open-source implementation of the Jakarta Servlet, Jakarta Pages (JSP), Jakarta Expression Language, and Jakarta WebSocket specifications[^1]. In plain terms it is the HTTP server plus servlet runtime that a large share of the Java web has run on since 1999, when Sun donated the original Java Servlet Development Kit codebase to the Apache Software Foundation[^2]. It is not a full Jakarta EE application server — there is no EJB, JMS, or CDI container in the box — and that deliberate narrowness is the reason it stayed small, stable, and everywhere.

The GitHub repository (~8.2k stars, ~5.4k forks) is a mirror; canonical development happens on ASF infrastructure with commits, releases, and bug tracking on Apache's own Git, mailing lists, and Bugzilla. The star count badly understates deployment reach — Tomcat predates GitHub, and most of its users never visit the repo. The high fork-to-star ratio reflects contributor and CI mirrors rather than typical popularity signals. It is actively maintained, with commits landing within days of any given week.

Tomcat's defining tension in the current era is the **`javax.*` → `jakarta.*` namespace migration**. When Oracle transferred Java EE to the Eclipse Foundation, trademark terms forbade further use of the `javax` package prefix, so Jakarta EE 9 renamed every servlet API package. Tomcat 10 adopted `jakarta.*`; Tomcat 9 and earlier stay on `javax.*`. The two are source- and binary-incompatible, and this single rename is the largest upgrade obstacle in the Java web ecosystem today[^3].

## Getting Started

Download a binary distribution, unpack, and run the startup script:

```bash
# Linux/macOS — Tomcat 11 requires Java 17+
curl -LO https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.0/bin/apache-tomcat-11.0.0.tar.gz
tar xzf apache-tomcat-11.0.0.tar.gz
cd apache-tomcat-11.0.0
./bin/catalina.sh run        # foreground; ./bin/startup.sh to daemonize
# app on http://localhost:8080/
```

A minimal servlet (Tomcat 10+, `jakarta.*` namespace):

```java
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;
import java.io.IOException;

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/plain");
        resp.getWriter().write("Hello from Tomcat");
    }
}
```

Package as a `.war`, drop it in `webapps/`, and Tomcat auto-deploys it. In practice most teams never do this by hand: Spring Boot embeds Tomcat and starts it from a plain `main()`, so the container is a library dependency, not a server you install.

## Architecture / How It Works

Tomcat is a hierarchy of nested components configured (in standalone mode) by `conf/server.xml`:

- **Server** → **Service** → **Connector**(s) + one **Engine** → **Host**(s) → **Context**(s). A Context is one deployed web application.
- **Coyote** is the connector layer — it parses HTTP and hands requests to the container. Connector implementations: **NIO** (default, non-blocking selectors), **NIO2** (async I/O), and historically **APR/native** (OpenSSL via JNI). The old blocking BIO connector was removed in 8.5.
- **Catalina** is the servlet container proper — the pipeline of Valves, the request/response lifecycle, session management, security.
- **Jasper** is the JSP engine: it compiles `.jsp` files to servlets on first request (or precompiled at build time).

**Class loading** is the part that surprises operators. Each web app gets its own classloader, isolated from other apps and inverted relative to the usual Java parent-first delegation (a web app's own classes win over shared ones, within limits). This isolation is what lets one Tomcat host multiple independent WARs — and it is also the source of the notorious **classloader memory leaks** on hot redeploy: `ThreadLocal`s, un-deregistered JDBC drivers, and lingering threads pin the old classloader in the heap. Tomcat ships listeners (`JreMemoryLeakPreventionListener`) and leak detection specifically to fight this, but a redeploy loop without a restart will eventually exhaust `Metaspace`.

The AJP connector (binary protocol for fronting Tomcat behind Apache httpd or nginx via `mod_jk` / `mod_proxy_ajp`) is disabled by default since the Ghostcat vulnerability (CVE-2020-1938) and now requires a shared `secret` and a bound address[^4].

## Production Notes

**Thread model.** Each Connector has a thread pool sized by `maxThreads` (default 200) with a `acceptCount` backlog and `connectionTimeout`. Under the default blocking servlet model, one request holds one thread for its full duration — slow downstream calls exhaust the pool. Async servlets (`AsyncContext`) and NIO2 relieve this but most apps run synchronously.

**Embedded vs standalone.** Since Spring Boot made embedded Tomcat the default, the operational surface shifted: no `server.xml`, no `webapps/` drop-in — configuration moves to `application.properties` and the container starts in-process. This is now the majority deployment mode, and standalone-Tomcat knowledge (Manager app, `catalina.sh`, `CATALINA_BASE` vs `CATALINA_HOME` split) is increasingly niche.

**Security footguns.** The `manager` and `host-manager` web apps are powerful and must never be exposed to the internet or left on default credentials — they are a standing remote-code-execution vector. The default `conf/` ships shutdown port 8005 listening on localhost with a plaintext `SHUTDOWN` command; lock it down or disable it. Keep up with `tomcat-announce` — Tomcat has a steady stream of CVEs typical of any long-lived server, and several (Ghostcat, request-smuggling, JSP-related) have been high-severity.

**The javax→jakarta upgrade.** Moving from Tomcat 9 to 10/11 is not a version bump; it is an API rename across every servlet import in your code and your dependencies. The Apache Tomcat Migration Tool for Jakarta EE rewrites bytecode/source, but any dependency still compiled against `javax.servlet` must be replaced or transformed. Plan this as a project, not a config change[^3].

**Runtime floors.** Tomcat 9 runs on Java 8+; Tomcat 10.1 requires Java 11+; Tomcat 11 requires Java 17+[^5]. Match the Tomcat major version to both your JDK and your target Jakarta EE profile before starting.

## When to Use / When Not

**Use when:**
- You run Servlet/JSP or Spring MVC applications and want the mature, boringly reliable reference container.
- You want an embeddable HTTP layer inside a Spring Boot / self-contained JAR.
- You need to host multiple independent WARs in one JVM with classloader isolation.
- You want an ASF-governed project with a long security-response track record and predictable major versions.

**Avoid when:**
- You need a full Jakarta EE server (EJB, JMS, CDI, JPA container) — Tomcat is a servlet container, not that.
- Your workload is reactive/non-blocking end to end — a Netty-based stack fits better than the servlet model.
- You want the smallest possible embedded footprint or fully async I/O by default — Jetty or Undertow are leaner.
- You want managed hosting with zero server ops — a serverless or PaaS runtime removes the container entirely.

## Alternatives

- eclipse/jetty — lighter, highly embeddable servlet container; use when footprint and embedding ergonomics matter more than being the spec reference.
- undertow-io/undertow — non-blocking, low-footprint web server from the WildFly project; use when you want async I/O and a smaller runtime (it is Spring Boot's alternate embedded server).
- wildfly/wildfly — full Jakarta EE application server; use when you actually need EJB/JMS/CDI, not just servlets.
- OpenLiberty/open-liberty — modular Jakarta EE / MicroProfile server; use when you want a full EE profile with fine-grained feature loading.
- netty/netty — low-level async network framework; use when you are building non-servlet reactive services and want direct control of the event loop.

## History

| Version | Date | Notes |
|---------|------|-------|
| 3.0 | 1999 | First Apache release; reference impl of Servlet 2.2 / JSP 1.1[^2]. |
| 4.0 | 2001 | Catalina rewrite of the container. |
| 6.0 | 2007 | Servlet 2.5 / JSP 2.1. |
| 7.0 | 2011 | Servlet 3.0, async support. |
| 8.5 | 2016 | BIO connector removed; long-lived LTS line. |
| 9.0 | 2018 | Servlet 4.0, HTTP/2; last `javax.*` major line[^1]. |
| 10.0 | 2020 | Jakarta EE 9 — `javax.*` → `jakarta.*` namespace break[^3]. |
| 10.1 | 2022 | Jakarta EE 10; Java 11+ required. |
| 11.0 | 2024 | Jakarta EE 11; Java 17+ required[^5]. |

## References

[^1]: Apache Tomcat homepage and specification versions. https://tomcat.apache.org/whichversion.html
[^2]: Apache Tomcat project history / "Get Involved" and heritage from the donated Sun servlet reference implementation. https://tomcat.apache.org/heritage.html
[^3]: Jakarta EE 9 namespace change and the Apache Tomcat Migration Tool for Jakarta EE. https://tomcat.apache.org/download-migration.cgi
[^4]: CVE-2020-1938 ("Ghostcat") — AJP connector file read/inclusion; AJP disabled by default and requires a secret thereafter. https://tomcat.apache.org/security-9.html
[^5]: Apache Tomcat version / Java version requirements matrix. https://tomcat.apache.org/whichversion.html

## Tags

java, servlet, jakarta-ee, web-server, http-server, jsp, application-server, spring-boot, network-server, apache
