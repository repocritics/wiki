# DigitalPlatDev/FreeDomain

A free domain-registration service operated as an OSS project — issue/PR flow handles domain requests, and the platform itself is open-sourced.

## What it is

A free domain platform run by the DigitalPlat organization that gives users subdomains under a set of domains the project owns (e.g. `*.digitalplat.org`). The GitHub repository is the operational surface: users open issues or PRs to claim or renew a subdomain, maintainers route them, and the platform's source code (HTML / Nginx config / scripts) is checked into the repo. AGPL-3.0 licensed — meaning anyone who runs a derivative service must also release their modifications.

## Key features

- Subdomain provisioning via GitHub issue / PR workflow rather than a web signup form.
- Multiple parent domains available — the project owns several memorable TLDs as the parent.
- Companion site at domain.digitalplat.org for the user-facing flow.
- AGPL-3.0 license — strong copyleft for the network-service case.
- HTML / shell scripts as the operational substrate.

## Tech stack

- HTML primary (the request-flow documentation + landing pages).
- Likely Nginx / static-site rendering for the user-facing surface (not surfaced in this excerpt).
- GitHub Issues + PRs as the application form.

## When to reach for it

- You want a free subdomain for a hobby project or open-source landing page without paying for DNS.
- You're a small team or individual developer comfortable with "issue-to-provision" flows.
- You're studying how a DNS-provisioning service can be operated entirely through GitHub issue automation.

## When *not* to reach for it

- You need a commercial domain with full DNS control and registrar transfer rights — this is a subdomain hand-out, not a TLD.
- You need SLAs — the project is volunteer-operated with no uptime guarantees.
- You're allergic to AGPL — if your stack consumes this code into a network-exposed service, the AGPL pulls your service into copyleft territory.

## Maturity signal

173k stars accumulated since May 2024 — high adoption for a service-of-this-kind. 3k forks is proportional to a service repo (forks are less useful when the value is the running service, not the code). Open issues low (33) reflects fast processing of domain requests, which is the main inbox. AGPL-3.0 + active maintenance + a real running service (domain.digitalplat.org) signal sustained operation rather than a parked project.

## Alternatives

- Cloudflare's free tier + a `$10/year` `.com` — use when you want full domain control.
- `is-a.dev` and similar GitHub-issue-based free-subdomain services — same model, different parent domain.
- `nip.io`, `sslip.io` — use when you only need IP-encoded subdomains for local development.

## Notes

The "OSS project that is also a service" pattern is unusual; verify the maintainers' parent-domain ownership and renewal practices before pointing anything mission-critical at a subdomain under it. AGPL is the right license for this — it forces any forks of the service to release their changes too. The high star count likely reflects users who starred to "save" the repo as a free-domain bookmark rather than to vouch for the code.

## Tags

free, domain, hosting, dns, agpl, html, self-hosted, awesome-list
