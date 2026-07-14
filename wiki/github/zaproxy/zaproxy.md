# zaproxy/zaproxy

> The Zed Attack Proxy — an open-source intercepting proxy and dynamic application security testing (DAST) scanner for finding vulnerabilities in running web apps.

[GitHub repo](https://github.com/zaproxy/zaproxy) ·
[Official website](https://www.zaproxy.org) ·
[License: Apache-2.0](https://github.com/zaproxy/zaproxy/blob/main/LICENSE)

## Overview

ZAP is a man-in-the-middle proxy that sits between a browser (or CI job) and a target web application, records the traffic, and then attacks it. It began in 2010 as a fork of the abandoned Paros Proxy, led by Simon Bennetts, and for over a decade was OWASP's flagship project. In 2023 the core team joined Checkmarx and the project left OWASP, rebranding to "ZAP by Checkmarx" — the code stayed Apache-2.0 and community-run, but the stewardship and funding moved to a commercial vendor[^1]. That transition is the single most important context for anyone evaluating ZAP today: it is still free and open, but no longer a vendor-neutral foundation project.

The tool serves two very different audiences from one codebase. For security engineers it is a point-and-click desktop app (Java/Swing) for manual, interactive testing — intercept a request, tamper with it, replay it, fuzz it. For DevSecOps it is a headless scanner driven from Docker and YAML that runs unattended in a pipeline. The defining tension is that DAST is inherently slow, noisy, and intrusive: unlike static analysis, ZAP has to actually send malicious traffic to a live application, which means false positives, long runtimes, and a real risk of damaging the target if pointed at the wrong environment.

ZAP competes directly with PortSwigger's Burp Suite. Burp is the paid industry standard for manual pentesting; ZAP is the OSS incumbent that is stronger in automation, CI integration, and scriptable/headless workflows, and weaker in polish and some advanced manual tooling.

## Getting Started

The most common entry point is the Docker packaged scan, not the desktop app:

```bash
# Passive-only baseline scan — safe, spiders + passively analyses, no attacks
docker run -t ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://www.example.com

# Full active scan — intrusive, sends attack payloads. Authorized targets only.
docker run -t ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py -t https://www.example.com
```

The current recommended way to configure non-trivial scans is the YAML Automation Framework rather than the older CLI flags:

```yaml
# automation.yaml — run with: zap.sh -cmd -autorun /zap/automation.yaml
env:
  contexts:
    - name: example
      urls: [https://www.example.com]
jobs:
  - type: spider
    parameters: { context: example }
  - type: activeScan
    parameters: { context: example }
  - type: report
    parameters: { template: traditional-html, reportFile: report.html }
```

The desktop UI is downloaded separately from zaproxy.org and requires a JRE (bundled installers are provided per platform).

## Architecture / How It Works

At its core ZAP is a proxy plus a set of scanners operating on the recorded traffic tree (the "Sites" tree):

- **Passive scanner** — inspects every request/response that flows through the proxy without modifying it. Cheap, safe, no extra requests. This is what the baseline scan uses.
- **Active scanner** — takes discovered endpoints and fires attack payloads (SQLi, XSS, path traversal, injection) at parameters. This is where the runtime, the false positives, and the risk live.
- **Spider** — a traditional HTML link crawler; and the **AJAX Spider**, which drives a real browser (via Selenium/WebDriver, built on Crawljax) to reach content behind JavaScript. The AJAX spider is necessary for SPAs but is heavy and slow.
- **Fuzzer, Requester, Breakpoints** — the manual-testing surface for tampering and replaying requests.

Almost all functionality ships as **add-ons** from the ZAP Marketplace rather than being in the core. The core repository is deliberately thin; scanners, report formats, API import (OpenAPI/SOAP/GraphQL), authentication helpers, and the HUD all live in the separate `zap-extensions` repo and are installed at runtime. This keeps releases decoupled but means a "ZAP install" is really core plus a fluid set of versioned add-ons whose compatibility you have to track.

Scripting is a first-class extension point: scan rules, authentication, payload generation and more can be written in JavaScript (GraalVM), and other engines (Python via Jython, Groovy, Kotlin) via add-ons. Everything the UI does is also exposed over a local **REST API**, which is how the Docker packaged scans and the Automation Framework drive the engine headlessly. Session and traffic data is persisted in an embedded HSQLDB database.

## Production Notes

**Active scanning is destructive by design.** The active scanner submits forms, follows links, and injects payloads — it can create records, trigger emails, delete data, or take a fragile app down. Never run a full/active scan against production or any target you are not explicitly authorized to attack. Baseline (passive) scans are the safe CI default.

**Runtime is unpredictable.** A full active scan of a large site can run for hours. The AJAX spider multiplies this because it launches real browsers. Teams typically time-box scans, scope them tightly with contexts, and disable expensive scan rules rather than let a pipeline run open-ended.

**Memory pressure.** ZAP keeps the site tree and session in the JVM heap (HSQLDB in-memory by default). Large crawls exhaust the heap; you will need to raise `-Xmx`, switch to a file-based session, or narrow scope. Long-running scans of big apps are the classic OOM scenario.

**Authentication is the hard part.** Getting ZAP to stay logged in — form-based, script-based, JSON/token, or the browser-based authentication add-on — is the most common source of frustration and of scans that silently only cover the login page. Budget real time to configure and verify authenticated scanning; a scan that lost its session produces a clean-looking but worthless report.

**False positives and triage.** DAST output requires human triage. ZAP alerts carry confidence levels for a reason; wiring raw ZAP results into a hard CI gate without a triage/baseline-suppression step produces noisy, ignored builds. The `-c` config-file and alert-filter add-ons exist to suppress known-benign findings.

**Add-on/version drift.** Because functionality is add-on-based and pulled from the Marketplace, reproducibility in CI depends on pinning the Docker image tag (`stable` vs `weekly` vs a specific version) and being aware that add-on updates can change scan behavior between runs.

## When to Use / When Not

**Use when:**
- You want free, automatable DAST in CI/CD (baseline scans on every build, scheduled full scans).
- You need a scriptable, headless, API-driven scanner rather than a GUI-only tool.
- You are doing manual web pentesting and want a no-cost intercepting proxy.
- You want to import an OpenAPI/GraphQL/SOAP definition and scan an API surface.

**Avoid when:**
- You need to scan production without risk — DAST attacks live traffic; use it against staging.
- You want zero-config, low-noise results — expect false positives and real triage effort.
- Manual pentesting polish and advanced tooling are the priority and budget exists — Burp Suite Professional is more refined here.
- You want SAST/dependency scanning — ZAP tests running apps, not source code or dependencies.

## Alternatives

- PortSwigger Burp Suite (proprietary, not on GitHub) — the commercial standard for interactive manual pentesting; choose it when manual testing is primary and budget allows.
- projectdiscovery/nuclei — fast template-based scanner; use instead when you want quick, low-false-positive checks for known CVEs and misconfigurations rather than deep crawl-and-attack.
- sullo/nikto — quick web-server and known-file misconfiguration scan; use for a fast first pass without a proxy.
- wapiti-scanner/wapiti — lightweight CLI black-box scanner; use when you want simple headless automation and no GUI.
- w3af/w3af — older Python web-app attack framework; historically ZAP's closest OSS peer, now largely inactive.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010 | Forked from Paros Proxy by Simon Bennetts; released as OWASP ZAP[^1]. |
| — | 2015-06 | `zaproxy/zaproxy` repository created on GitHub. |
| 2.x | 2013–present | Long-running 2.x line; add-on/Marketplace model matures. |
| — | 2023 | ZAP core team joins Checkmarx; project leaves OWASP, rebrands "ZAP by Checkmarx"[^1]. |
| 2.16 | 2024–2026 | Current 2.16.x series; Automation Framework is the recommended scan-configuration path[^2]. |

## References

[^1]: "ZAP is joining Checkmarx" — ZAP blog, 2023. https://www.zaproxy.org/blog/2023-08-01-zap-is-joining-checkmarx/
[^2]: ZAP Automation Framework documentation. https://www.zaproxy.org/docs/automate/automation-framework/

## Tags

java, security, dast, web-security, penetration-testing, security-scanner, appsec, proxy, vulnerability-scanner, devsecops, owasp
