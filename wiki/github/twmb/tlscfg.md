# twmb/tlscfg

> Option-driven Go package for building a `*tls.Config` with secure defaults instead of hand-rolling one per project.

[GitHub repo](https://github.com/twmb/tlscfg) ·
[License: BSD-3-Clause](https://github.com/twmb/tlscfg/blob/main/LICENSE)

## Overview

`tlscfg` is a small Go library that wraps the rote, error-prone work of
constructing a `crypto/tls.Config`. Standard-library TLS setup is verbose:
you load PEM files from disk, parse them into `tls.Certificate` and
`x509.CertPool` values, wire them into the right config fields, and pick
cipher/version defaults yourself. The defaults that ship in `crypto/tls` are
reasonable but not obviously so, and the assembly code is easy to get subtly
wrong. `tlscfg` collapses that into a single `New(...Opt)` call with a functional-options
API.

The package is authored by Travis Bischel (twmb), better known for the
`franz-go` Kafka client, and it exists largely to serve that kind of
networked-client code where mutual TLS against a custom CA is common. It is
deliberately narrow: it does not manage certificate rotation, does not run an
ACME client, and does not abstract away `*tls.Config` — it hands you a normal
one that you can still mutate. The defining tradeoff is scope. This is a
convenience layer, not a framework, and its value is proportional to how much
boilerplate TLS wiring your codebase would otherwise repeat.

It is a low-star, low-traffic utility[^1] — the kind of dependency you adopt
because you trust the author and want to stop copy-pasting cert-loading code,
not because of a large community around it. Treat it as vendored convenience
rather than critical infrastructure.

## Getting Started

```bash
go get github.com/twmb/tlscfg
```

```go
package main

import (
	"crypto/tls"
	"log"

	"github.com/twmb/tlscfg"
)

func main() {
	cfg, err := tlscfg.New(
		tlscfg.MaybeWithDiskCA("ca.pem", tlscfg.ForClient), // optional custom CA
		tlscfg.WithDiskKeyPair("cert.pem", "key.pem"),      // client cert + key
		tlscfg.WithSystemCertPool(),                        // add system roots
	)
	if err != nil {
		log.Fatal(err)
	}

	_ = &tls.Config{} // cfg is a normal *tls.Config you can mutate further
	_ = cfg
}
```

`New` returns a config seeded with system certificates and TLS 1.2+ ciphers;
the `With*` options layer additional certificates or override settings.

## Architecture / How It Works

The whole package is a functional-options constructor. `New(opts ...Opt)`
starts from a baseline `*tls.Config` (minimum version TLS 1.2, a curated
cipher list) and applies each option in order, returning the config or the
first error encountered. Options are ordinary closures over the config, so
ordering is significant when two options touch the same field.

The option set covers the common cases: load a CA certificate from disk or
from an in-memory PEM byte slice; load a leaf key pair from disk or memory;
attach the system certificate pool. CA options take a direction argument
(`ForClient` / `ForServer`) that selects whether the pool becomes
`RootCAs` (verifying a server) or `ClientCAs` (verifying clients). The
`Maybe*` variants no-op when their path argument is empty, which is what lets
you drive optional mTLS straight from a possibly-unset CLI flag without an
`if` around the option.

Because the result is a plain `*tls.Config`, anything the package does not
model — `ServerName`, `InsecureSkipVerify`, `NextProtos`, custom
`VerifyPeerCertificate` hooks — is set by mutating the returned struct
directly. The library intentionally stops at construction and does not try to
own the config's lifetime.

## Production Notes

- **No rotation.** `tlscfg` reads certificate files once, at construction. If
  your leaf certs rotate on disk (cert-manager, Vault agent, ACME renewals),
  the returned `*tls.Config` keeps serving the certs it loaded at startup. For
  hot reload you need `GetCertificate` / `GetClientCertificate` callbacks,
  which you must wire yourself after `New` returns.
- **Errors are first-error, at build time.** A missing or malformed PEM file
  surfaces as an error from `New`, not at connection time — good for
  fail-fast startup, but it means you should call `New` during initialization,
  not lazily on first request.
- **Defaults are a point-in-time opinion.** The baseline TLS version and
  cipher selection reflect the maintainer's view when last updated (last
  pushed early 2026[^1]). If you have specific compliance requirements
  (FIPS, a mandated cipher suite, TLS 1.3-only), verify and override the
  returned config rather than assuming the defaults match your policy.
- **Small surface, small blast radius, small support base.** One maintainer,
  effectively no open issues, minimal external usage. That is fine for a
  ~single-file helper you can read end to end, but do read it — you are
  trusting one person's cert-loading code in your TLS path. Vendoring or
  pinning a specific tag is prudent.
- **Not a server framework.** It builds a config; listening, `http.Server`
  wiring, ALPN, and graceful shutdown are all still yours.

## When to Use / When Not

**Use when:**
- You repeatedly build `*tls.Config` for Go network clients/servers with a
  custom CA and/or client certificates and want to stop copy-pasting the
  loader code.
- You want optional mTLS driven directly from config flags (the `Maybe*`
  options).
- You value a small, readable dependency you can audit in one sitting.

**Avoid when:**
- You need certificate hot-reload / rotation without a restart.
- You want ACME/Let's Encrypt automation (use `autocert`).
- You need a fully managed HTTPS server story rather than just a config.
- Your org forbids small single-maintainer dependencies in the security path.

## Alternatives

- golang/go `crypto/tls` — the standard library; write the ~20 lines yourself when you want zero dependencies or full control.
- caddyserver/certmagic — use instead when you want automatic issuance, storage, and renewal of certificates (ACME) rather than loading your own.
- golang/crypto (`acme/autocert`) — use for on-demand Let's Encrypt certificates for public-facing HTTPS servers.
- spiffe/go-spiffe — use when identities come from a SPIFFE/SPIRE workload API and you need rotating SVIDs, not static PEM files.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-08-17 | Repository created; option-driven `*tls.Config` builder[^1]. |
| latest push | 2026-02-27 | Most recent commit as of this writing; low but non-abandoned activity[^1]. |

Tagged releases exist on the Go module proxy; consult the repo's tags for
exact version numbers before pinning[^2].

## References

[^1]: GitHub API metadata for `twmb/tlscfg` — Go, BSD-3-Clause, created 2021-08-17, last pushed 2026-02-27, ~8 stars / 1 fork, 0 open issues. https://github.com/twmb/tlscfg
[^2]: Go package documentation for the module API (`New`, `With*` / `Maybe*` options, `ForClient` / `ForServer`). https://pkg.go.dev/github.com/twmb/tlscfg

## Tags

go, golang, tls, crypto, security, certificates, mtls, x509, networking, library, functional-options
