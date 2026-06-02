# KeygraphHQ/shannon

Shannon Lite — an autonomous white-box AI pentester for web applications and APIs. Analyzes source code, identifies attack vectors, executes real exploits.

## What it is

A TypeScript security automation tool that combines source-code analysis with active exploitation against running services. Marketed as a "white-box AI pentester" — it has access to your code (white-box) and runs real exploits (active testing). AGPL-3.0 licensed. Distributed by Keygraph (keygraph.io) as the open-source variant of their commercial security platform.

## Key features

- White-box analysis — reads source code to model the attack surface.
- Active exploitation — runs real attacks against the target service to validate vulnerabilities.
- API + web-app coverage.
- TypeScript implementation.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary.

## When to reach for it

- You're a security engineer running pentests against your own services and want AI-augmented automation.
- You're evaluating AI-pentest tooling and want a hands-on look.

## When *not* to reach for it

- You don't have authorization to pentest the target — running active exploits against systems you don't own is illegal.
- You want vendor-supported tooling with SLAs — Keygraph's commercial offering is the closer-fit path.
- AGPL-3.0 doesn't fit your commercial model.

## Maturity signal

44k stars, 5k forks, AGPL-3.0. Open-issues count of 10 is low. The security-automation space requires careful authorization controls; verify your scope before running active exploits.

## Alternatives

- Burp Suite — commercial industry standard for web pentest.
- OWASP ZAP — OSS web application security scanner.
- Nuclei (ProjectDiscovery) — template-based vulnerability scanner.

## Tags

typescript, security, penetration-testing, agpl, automation, ai, security-tools
