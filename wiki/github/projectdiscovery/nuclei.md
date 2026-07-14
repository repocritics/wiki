# projectdiscovery/nuclei

> A template-driven vulnerability scanner: security checks are YAML files, not code, so the community writes the detections and the engine just runs them fast.

[GitHub repo](https://github.com/projectdiscovery/nuclei) ·
[Official website](https://docs.projectdiscovery.io/tools/nuclei) ·
[License: MIT](https://github.com/projectdiscovery/nuclei/blob/dev/LICENSE.md)

## Overview

Nuclei is a Go-based vulnerability scanner from ProjectDiscovery, first released in 2020[^1]. Its central design decision is to move detection logic out of the scanner and into declarative YAML templates: each template describes a request to send (HTTP, DNS, TCP, SSL, headless-browser, JavaScript, WHOIS, etc.) and a set of matchers/extractors that decide whether the response indicates a vulnerability. The engine itself is a request scheduler and matcher runtime; the "what to look for" lives in the separate [nuclei-templates](https://github.com/projectdiscovery/nuclei-templates) repository, which the community maintains and which nuclei downloads and updates on its own.

This split is the whole value proposition and also the whole tension. Because detections are data, a new CVE can get a working check within hours of disclosure without recompiling anything, and thousands of contributors can add coverage. The cost is that scan quality is only as good as the templates you run: a template with a loose matcher produces false positives, and the "zero false positives" framing in the project's marketing is an aspiration of the template-authoring discipline, not a guarantee of the engine. Nuclei is best understood as infrastructure for running a corpus of checks, not as an opinionated product that tells you what your risk is.

It is widely used in bug-bounty reconnaissance, external attack-surface monitoring, and CI/CD security regression testing. ProjectDiscovery also sells a hosted Pro/Enterprise platform built on the OSS engine[^2]; the open-source CLI remains fully functional and MIT-licensed, but some scale and reporting features live only in the paid tier.

## Getting Started

Install via Go (requires Go >= 1.24.2)[^3], or a prebuilt release binary / Docker image:

```sh
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

Scan a single target with the default community templates (auto-downloaded on first run):

```sh
nuclei -u https://example.com
```

Run a specific template set and write JSONL for downstream tooling:

```sh
nuclei -l hosts.txt -t http/cves/ -severity critical,high -j -o findings.jsonl
```

A minimal template — the actual unit of work in nuclei:

```yaml
id: example-exposed-env
info:
  name: Exposed .env file
  author: you
  severity: high
http:
  - method: GET
    path:
      - "{{BaseURL}}/.env"
    matchers-condition: and
    matchers:
      - type: status
        status: [200]
      - type: word
        words: ["DB_PASSWORD", "APP_KEY"]
        condition: or
```

## Architecture / How It Works

Nuclei's runtime is organized around **protocols** and **operators**. A template declares one or more protocol blocks (`http`, `dns`, `tcp`, `ssl`, `headless`, `javascript`, `code`, `whois`, `websocket`, `file`). Each block produces requests; the responses are fed to **matchers** (status, word, regex, binary, DSL expressions) and **extractors** (regex, kval, JSON, XPath) that decide detection and pull out data. Matcher/extractor logic is shared across all protocols, which is why the same mental model transfers from an HTTP CVE check to a TLS misconfiguration check.

Performance comes from two mechanisms. First, heavy parallelism: templates run concurrently (`-c`, default 25), each template analyzes multiple hosts in parallel (`-bs`, default 25), and requests are rate-limited globally (`-rl`, default 150/s). Second, **request clustering** — when many templates send the identical HTTP request (same method/path/headers) and differ only in their matchers, nuclei sends the request once and runs all the matchers against the single response. On large CVE runs this collapses thousands of redundant requests and is a major reason nuclei scales; it is also why `-resume` disables clustering, since resumption needs deterministic per-template request accounting.

Out-of-band detection uses **Interactsh**[^4], ProjectDiscovery's OAST server. Templates that inject a callback URL (blind SSRF, blind XXE, some RCEs) point at a public Interactsh server by default (`oast.pro` and siblings) and poll it for interactions. This means blind checks phone home to third-party infrastructure unless you self-host Interactsh with `-iserver`/`-itoken` — a real consideration for scanning sensitive internal assets.

Templates can be **signed**. Since the `code` protocol and self-contained templates can execute arbitrary logic on the scanning host, nuclei enforces signature verification: official templates are signed with ProjectDiscovery's key, and unsigned/tampered templates are flagged or blocked. Loading the `code` protocol requires the explicit `-code` flag, and `-disable-unsigned-templates` hardens this further. The threat model is that templates are executable content, not inert config.

## Production Notes

- **Templates are the moving part, not the binary.** Nuclei auto-updates `nuclei-templates` on run. That means your scan results change when the community ships template changes, even if you pinned the nuclei version. For reproducible CI, pin the templates directory (vendored copy or `-ud`/`-duc` to control update behavior) rather than relying on the auto-updated default.
- **Default rate limits are conservative but still dangerous.** 150 req/s across a fleet of hosts can knock over fragile targets, trip WAFs, or generate alert-storms for a blue team. Tune `-rl`, `-c`, and `-bs` per engagement; on production targets start low.
- **Severity counts are not risk.** A run reporting 400 "info" findings and 3 "critical" is normal; the info tier is mostly technology fingerprinting. Filter with `-severity` and treat template `severity` as author-assigned metadata, not a validated CVSS.
- **False positives track template quality.** Broadly-tagged or old templates can match on error pages, honeypots, or generic banners. When a finding matters, read the template's matcher and reproduce the request manually before reporting.
- **OAST leakage.** Blind/OAST templates contact public Interactsh infrastructure by default. For internal or regulated scanning, self-host Interactsh or exclude OAST templates with `-ni`.
- **Breaking changes between releases.** The project states it is in active development and expects breaking changes; the CLI flag surface and template schema evolve, and the module path is versioned (`/v3`). Review the changelog before upgrading in an automated pipeline.
- **Running as a service is discouraged.** The maintainers explicitly warn against exposing nuclei as a long-running network service; it is designed as a CLI tool, and the experimental HTTP API endpoint (`-hae`) is not a hardened surface.
- **Headless and code protocols pull in weight.** Headless templates require a Chromium install and are far slower; the `code` protocol executes host-side and is off by default for good reason. Keep them opt-in.

## When to Use / When Not

**Use when:**
- You want fast, scriptable detection of known CVEs and misconfigurations across many hosts.
- You're doing external attack-surface monitoring or bug-bounty recon and want community coverage of trending vulns.
- You need security regression checks wired into CI/CD with machine-readable (JSONL/SARIF) output.
- You want to codify org-specific checks as YAML your team can review and version.

**Avoid when:**
- You need authenticated, stateful application security testing with deep crawl/auth handling — nuclei's DAST/fuzzing is real but not a full DAST product.
- You need software composition analysis or SAST — nuclei scans running systems, not source or dependency trees.
- You require guaranteed-accurate risk scoring out of the box — template quality varies and results need triage.
- You cannot allow blind checks to contact third-party OAST infrastructure and won't self-host Interactsh.

## Alternatives

- OWASP/Nettacker — open-source recon + vulnerability scanner; broader modules, less template-community momentum. Use when you want an all-in-one recon toolkit rather than a template engine.
- zaproxy/zaproxy — full DAST proxy with crawling, auth, and active scan. Use when you need authenticated, spidered web-app testing rather than fast signature checks.
- greenbone/openvas-scanner — network vulnerability scanner with a large NVT feed. Use for classic infra/CVE scanning of internal networks with an established feed model.
- future-architect/vuls — agentless host CVE scanner driven by package inventory. Use when you want OS/package-level CVE detection rather than network-response detection.
- sullo/nikto — long-standing web server scanner. Use for quick, dependency-light web server checks when you don't need nuclei's protocol breadth.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2020-04 | Initial release; YAML-template HTTP scanner[^1]. |
| v2 | 2020 | Matured template schema, added protocols (DNS, network, file). |
| v2.5+ | 2021–2022 | Interactsh/OAST integration, headless and workflow support. |
| v3.0 | 2023-08 | Major release; module path `/v3`, code protocol, JS protocol, template signing[^5]. |
| v3.x | 2024–2026 | DAST/fuzzing (`-dast`), AI prompt template generation (`-ai`), OpenAPI/Swagger input modes, ongoing template-schema evolution. |

## References

[^1]: ProjectDiscovery, "Nuclei" project on GitHub — repository created 2020-04-03. https://github.com/projectdiscovery/nuclei
[^2]: ProjectDiscovery Pro/Enterprise (cloud platform built on the Nuclei engine). https://projectdiscovery.io/pricing
[^3]: Nuclei installation guide (Go version requirement). https://docs.projectdiscovery.io/tools/nuclei/install
[^4]: Interactsh — OAST (out-of-band application security testing) infrastructure used for blind detection. https://github.com/projectdiscovery/interactsh
[^5]: Nuclei documentation and releases. https://docs.projectdiscovery.io/tools/nuclei/overview
[^6]: nuclei-templates — the community template corpus the engine runs. https://github.com/projectdiscovery/nuclei-templates

## Tags

go, security, vulnerability-scanner, dast, cve-scanner, yaml-templates, attack-surface, cli, devsecops, oast, penetration-testing
