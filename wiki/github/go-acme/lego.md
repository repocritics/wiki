# go-acme/lego

> ACME (Let's Encrypt) client and Go library — a one-shot certificate tool plus 200+ DNS providers, not a long-running HTTPS daemon.

[GitHub repo](https://github.com/go-acme/lego) ·
[Official website](https://go-acme.github.io/lego/) ·
[License: MIT](https://github.com/go-acme/lego/blob/master/LICENSE)

## Overview

lego is an ACME client and library written in Go, used to obtain, renew, and revoke TLS certificates from Let's Encrypt and any other CA that speaks ACME (RFC 8555)[^2]. It started in 2015 as `xenolf/lego` — one of the earliest third-party ACME clients — and was later transferred to the community-run `go-acme` organization[^1]. It ships in two shapes from the same codebase: a standalone `lego` CLI and an importable Go package (`github.com/go-acme/lego/v5`).

The defining characteristic is scope. Unlike Caddy or cert-manager, lego does not run as a daemon and holds no reconciliation loop: it issues or renews a certificate when you invoke it and then exits. Scheduling (cron, systemd timers, a wrapping service) is the operator's job. This makes lego small and composable — it is the ACME engine embedded inside Traefik[^7] and countless bespoke tools — but it means "automatic HTTPS" is something you assemble, not something you get for free.

The second defining characteristic is the DNS provider sprawl. lego supports the DNS-01 challenge against more than 200 DNS APIs[^3], each a separate community-contributed package configured through environment variables. That breadth is the main reason to reach for lego over alternatives — but the long tail of providers varies in maintenance quality and propagation behavior, and that variance is where most real-world failures live.

## Getting Started

```bash
# CLI: install the binary (module path is versioned — note /v5)
go install github.com/go-acme/lego/v5/cmd/lego@latest
# also available via Homebrew (brew install lego) and Docker (goacme/lego)

# HTTP-01 challenge, standalone listener on :80
lego --email you@example.com --domains example.com --http run

# DNS-01 with Cloudflare — required for wildcards
CLOUDFLARE_DNS_API_TOKEN=xxxxx \
  lego --email you@example.com --domains '*.example.com' --dns cloudflare run
```

```go
// Library: myUser must implement registration.User
// (GetEmail / GetRegistration / GetPrivateKey).
config := lego.NewConfig(myUser)
config.CADirURL = lego.LEDirectoryProduction // use LEDirectoryStaging while testing
config.Certificate.KeyType = certcrypto.EC256

client, _ := lego.NewClient(config)

provider, _ := cloudflare.NewDNSProvider()          // providers/dns/cloudflare
client.Challenge.SetDNS01Provider(provider)

reg, _ := client.Registration.Register(
	registration.RegisterOptions{TermsOfServiceAgreed: true})
myUser.Registration = reg

res, _ := client.Certificate.Obtain(certificate.ObtainRequest{
	Domains: []string{"*.example.com", "example.com"},
	Bundle:  true, // res.Certificate is leaf + issuer chain
})
```

## Architecture / How It Works

lego is layered: a thin ACME protocol client (`acme/api`) implementing account registration, order/authorization/challenge flows, and certificate finalization; a set of challenge solvers; and the CLI and library facades on top. The CLI is a wrapper over the same public library, so anything the tool does can be done programmatically.

Certificates are obtained by solving one of three ACME challenges. **HTTP-01** serves a token at `/.well-known/acme-challenge/` — either via lego's own standalone listener on port 80 or by writing into a webroot. **DNS-01** provisions a `_acme-challenge` TXT record through a provider API; it is the only challenge that can issue wildcards and the only one that works when the host is not publicly reachable on 80/443. **TLS-ALPN-01** answers on port 443 using the ALPN extension (RFC 8737). DNS-01 also supports CNAME delegation, so the validated TXT record can live in a separate, dedicated zone.

For DNS-01, lego does more than call an API: after writing the TXT record it polls authoritative nameservers until the value has propagated, and only then tells the CA to validate. Each provider package declares its own propagation and polling defaults. Account keys, registration data, and issued certificates are persisted to disk (`~/.lego` or `--path` for the CLI); the library leaves storage to the caller.

Recent protocol work includes ARI — ACME Renewal Information (RFC 9773)[^6] — which lets the CA suggest renewal windows, plus support for IP-address certificates (RFC 8738) and draft profiles/persistent-DNS extensions. The Go module uses semantic import versioning, so each major version lives at a distinct import path (`/v2` … `/v5`)[^5].

## Production Notes

**DNS propagation is the number-one failure mode.** A correct API call does not mean the record is visible to the CA's resolvers. Slow-propagating providers, split-horizon DNS, and aggressive negative caching all cause validation to time out. Tune the propagation/polling settings for your provider (CLI flags like `--dns.propagation-wait`, or disabling lego's own precheck and letting the CA poll) rather than assuming the defaults fit.

**Rate limits are the CA's, not lego's.** Let's Encrypt enforces per-registered-domain issuance limits and duplicate-certificate limits. Loop your automation against the staging directory (`LEDirectoryStaging`) first — burning production quota on a misconfiguration means waiting out a rolling window.

**There is no renewal daemon.** `lego ... renew --days 30` is a one-shot that only acts if the certificate is within the threshold; you must schedule it yourself and reload the consuming service afterward. Teams that want fire-and-forget behavior often end up wrapping lego, at which point Caddy or cert-manager may be the better fit.

**Major-version upgrades change the import path.** Because of Go's semantic import versioning, moving from `/v4` to `/v5` is an import rewrite across your codebase, sometimes with accompanying API changes — not a transparent `go get -u`. The CLI hides this, but library consumers feel it at each major bump.

**Operational hygiene:** HTTP-01 needs inbound port 80 (redirects and some proxies break it); wildcards force DNS-01; back up account and certificate keys since they live as plain files; and treat provider credentials in environment variables as secrets, since a compromised DNS token allows challenge forgery for the whole zone.

## When to Use / When Not

**Use when:**
- You need ACME issuance inside a Go program, or a scriptable CLI for cron/systemd.
- You need DNS-01 against a specific provider or wildcards — lego's provider coverage is the widest available.
- You want a small, embeddable engine you control, not an opinionated always-on server.

**Avoid when:**
- You want zero-config, always-on HTTPS — a server like Caddy manages renewal for you.
- You run Kubernetes — cert-manager models Certificates as CRDs with controllers.
- You just need certs for nginx/apache on a standard host — certbot's server plugins are more batteries-included.

## Alternatives

- certbot/certbot — EFF's Python client with nginx/apache plugins; use it for a conventional single-server web stack.
- caddyserver/certmagic — the library behind Caddy's automatic HTTPS; use it when you want a long-running server to own renewal.
- cert-manager/cert-manager — Kubernetes-native issuance via CRDs; use it inside a cluster.
- acmesh-official/acme.sh — pure-shell client with broad DNS support; use it when you want no compiled runtime or language dependency.
- smallstep/certificates — run your own ACME CA (step-ca); use it for internal PKI rather than public certs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015 | Initial release as `xenolf/lego`; ACME v1 / Let's Encrypt beta[^1]. |
| 1.0.0 | 2018-05-31 | First stable tag. |
| 2.0.0 | 2019-01-09 | Rewrite for ACME v2 / RFC 8555; API restructured, import path `/v2`[^4]. |
| 3.0.0 | 2019-08-07 | Major release, import path `/v3`. |
| 4.0.0 | 2020-09-02 | Major release, import path `/v4`. |
| 5.0.0 | 2026-05-11 | Current major line, import path `/v5`; ARI (RFC 9773) and newer extensions. |

## References

[^1]: lego repository (originally `xenolf/lego`, now `go-acme/lego`). https://github.com/go-acme/lego
[^2]: RFC 8555 — Automatic Certificate Management Environment (ACME). https://www.rfc-editor.org/rfc/rfc8555.html
[^3]: lego DNS provider documentation. https://go-acme.github.io/lego/dns/
[^4]: lego v2.0.0 release (ACME v2). https://github.com/go-acme/lego/releases/tag/v2.0.0
[^5]: Go modules — major version suffixes (semantic import versioning). https://go.dev/ref/mod#major-version-suffixes
[^6]: RFC 9773 — ACME Renewal Information (ARI) Extension. https://www.rfc-editor.org/rfc/rfc9773.html
[^7]: Traefik documentation — ACME/Let's Encrypt (built on lego). https://doc.traefik.io/traefik/https/acme/

## Tags

go, acme, letsencrypt, tls, certificates, dns-01, x509, cli, library, pki, security
