# smallstep/certificates

> An online private certificate authority (X.509 + SSH) and ACME server for automated, short-lived certificate management in DevOps.

[GitHub repo](https://github.com/smallstep/certificates) ·
[Official website](https://smallstep.com/certificates) ·
[License: Apache-2.0](https://github.com/smallstep/certificates/blob/master/LICENSE)

## Overview

`step-ca` is a self-hosted certificate authority written in Go, maintained by Smallstep Labs since it was open-sourced in 2018[^1]. It issues X.509 TLS certificates (for services, VMs, containers, Kubernetes pods, mTLS between microservices) and SSH certificates (for users and hosts), and it speaks ACME — the same protocol Let's Encrypt uses — so existing clients like certbot, acme.sh, Caddy, and Traefik can request certificates from your own internal CA[^2]. It is the server counterpart to the `step` CLI, and the two are usually deployed together.

The defining design choice is **short-lived certificates with passive revocation**. Rather than publishing CRLs or running OCSP responders, the open-source `step-ca` issues certificates with short lifetimes (24h is the default) and expects automated renewal[^3]. A compromised key stops being trusted when it expires, not when a revocation list propagates. This is a genuinely different operational model from traditional PKI, and whether it fits depends entirely on whether you can automate renewal everywhere.

The project is explicitly positioned as the free tier beneath Smallstep's commercial Certificate Manager. The README is candid that several capabilities operators eventually want — multiple CAs, active CRL/OCSP revocation, high-availability turnkey deployment, ACME External Account Binding, web admin UI, FIPS builds — are reserved for the paid product[^4]. `step-ca` is fully usable and widely deployed as-is, but the boundary is real and worth understanding before you build on it.

## Getting Started

```bash
# macOS (installs both step-ca and the step CLI)
brew install step

# Initialize a new CA (interactive: names, provisioner, password)
step ca init

# Start the CA using the generated config
step-ca $(step path)/config/ca.json
```

```bash
# From another machine: bootstrap trust, then request a cert
step ca bootstrap --ca-url https://ca.internal:9000 \
  --fingerprint <root-fingerprint>

step ca certificate svc.internal svc.crt svc.key
# → svc.crt / svc.key, auto-renewable via `step ca renew`
```

Docker is the other common path: `docker run -d -p 9000:9000 -v step:/home/step smallstep/step-ca`.

## Architecture / How It Works

`step-ca` is a single Go binary exposing an HTTPS API (default port 9000). The core abstraction is the **provisioner**: a configured method for authorizing a certificate request and mapping an identity to a certificate. Provisioner types include ACME, OIDC (OAuth ID tokens from Okta, Google, Azure AD, Keycloak, Dex), cloud instance identity documents (AWS/GCP/Azure VM attestation), single-use JWK tokens (for Terraform/Ansible/Chef CD pipelines), X5C (present an existing X.509 cert), SCEP, Nebula, and SSHPOP (renew an SSH host cert with the cert itself)[^5]. Each provisioner carries its own policy — allowed SANs, certificate lifetimes, key types.

Certificate templates are Go `text/template` documents that render the final X.509 or SSH certificate, giving fine control over extensions, key usages, and name constraints without patching the server. Signing keys can live on disk or in a KMS: `step-ca` ships integrations for PKCS#11 HSMs, YubiKey PIV, and cloud KMS backends (AWS KMS, Google Cloud KMS, Azure Key Vault)[^6], so the CA private key need not sit in a file.

Persistence is pluggable: Badger (default, embedded), BoltDB, PostgreSQL, and MySQL[^7]. The database stores issued-certificate records, ACME order state, and provisioner data. The recommended topology is two-tier — an offline root CA whose key is used once to sign an intermediate, and `step-ca` running as the online intermediate that does all day-to-day issuance. The root key can then stay air-gapped.

The ACME implementation covers `http-01`, `dns-01`, and `tls-alpn-01` challenges over ACMEv2 (RFC 8555), which is what lets unmodified public-CA clients point at an internal endpoint.

## Production Notes

**No active revocation in open source.** This is the single most important caveat. CRL and OCSP are commercial-only[^4]; the open-source answer to "revoke this cert now" is "make lifetimes short and renew often." If a regulatory or security requirement mandates active revocation with immediate propagation, `step-ca` alone does not satisfy it. `step ca revoke` exists but relies on the passive/short-lived model for most deployments.

**Single CA, and it's the online signing path.** One `step-ca` process serves one CA. There is no built-in multi-tenant CA hierarchy in the open-source build. Because the intermediate key is online to sign requests, protect it accordingly — KMS/HSM backing is strongly recommended for anything beyond a lab.

**High availability is your problem.** Open-source `step-ca` has no turnkey HA. Running multiple replicas requires a shared SQL database (Postgres/MySQL, not Badger/Bolt) and a load balancer; there is no clustering layer. Renewal traffic is bursty by nature — every short-lived cert in your fleet re-hits the CA on its own schedule — so size for the aggregate renewal rate, not the issuance count.

**Renewal must actually run everywhere.** The short-lived model is only safe if every consumer renews reliably. A workload that fails to renew before expiry simply breaks. Teams typically run `step` as a sidecar, a systemd timer, cert-manager (which supports `step-ca` as an ACME/StepIssuer backend), or a Caddy/Traefik integration. An un-automated corner of the fleet becomes an outage waiting to happen.

**Still pre-1.0.** Despite years of production use and thousands of deployments, releases remain `v0.x`. Config format and provisioner behavior have been stable in practice, but the version number signals the maintainers reserve the right to change things; read release notes before upgrading.

## When to Use / When Not

**Use when:**
- You need an internal/private CA for mTLS, service identity, or SSH certificates and want automation, not manual `openssl` PKI.
- You already run ACME clients (Caddy, certbot, cert-manager) and want to point them at internal infrastructure.
- Short-lived certificates with automated renewal fit your environment.
- You want SSO-backed SSH or TLS issuance via OIDC without building it yourself.

**Avoid when:**
- You require active CRL/OCSP revocation, multiple CAs, HA out of the box, EAB, or FIPS builds — those push you to the commercial Certificate Manager or another product.
- You cannot guarantee automated renewal across the whole fleet.
- You need a secrets platform, not just a CA (Vault covers broader ground).
- You want a long-term-frozen 1.0 API contract today.

## Alternatives

- cloudflare/cfssl — CFSSL is a PKI toolkit and library with a signing API, but has no built-in ACME server or provisioner model; use it when you want a lower-level toolkit to embed rather than a running CA.
- hashicorp/vault — Vault's PKI secrets engine issues certificates as one feature of a broader secrets platform; use it when you already run Vault and want certs alongside secrets, dynamic credentials, and transit encryption.
- cert-manager/cert-manager — Kubernetes-native certificate controller; use it inside a cluster, often pointed *at* `step-ca` as the issuing backend rather than as a replacement.
- letsencrypt/boulder — the software behind Let's Encrypt; use it when you are operating a large public-facing ACME CA, not a small internal one.
- OpenVPN/easy-rsa — scripts around OpenSSL for manual PKI; use it only for tiny static setups where automation is not needed.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-11 | `step-ca` open-sourced by Smallstep; X.509 issuance + provisioners[^1]. |
| SSH CA | 2019 | SSH user and host certificate issuance added[^8]. |
| ACME server | 2020 | ACMEv2 (RFC 8555) server support for standard clients[^2]. |
| KMS/HSM | 2020–2021 | PKCS#11, YubiKey, and cloud KMS signing-key backends[^6]. |
| ongoing | 2026 | Actively maintained (commits within the last day at time of writing); still versioned `v0.x`. |

## References

[^1]: smallstep/certificates repository, created 2018-11-01. https://github.com/smallstep/certificates
[^2]: "Private ACME server" / ACME basics, Smallstep docs. https://smallstep.com/docs/step-ca/acme-basics/
[^3]: "Passive revocation" and short-lived certificates, Smallstep blog. https://smallstep.com/blog/passive-revocation.html
[^4]: README "Comparison with Smallstep's commercial product" — lists multiple CAs, active revocation (CRL/OCSP), HA, EAB, FIPS, HSM-bound keys, and web admin UI as commercial features. https://github.com/smallstep/certificates#readme
[^5]: Provisioner documentation, Smallstep. https://smallstep.com/docs/step-ca/provisioners
[^6]: Cryptographic protection / KMS integrations (PKCS#11, YubiKey, AWS/GCP/Azure KMS), Smallstep docs. https://smallstep.com/docs/step-ca/cryptographic-protection
[^7]: Database backends (Badger, BoltDB, PostgreSQL, MySQL), Smallstep configuration docs. https://smallstep.com/docs/step-ca/configuration#databases
[^8]: SSH certificates with step-ca, Smallstep blog. https://smallstep.com/blog/use-ssh-certificates/

## Tags

go, pki, certificate-authority, x509, tls, ssh, acme, security, devops, mtls, self-hosted
