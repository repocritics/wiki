# hashicorp/vault

> Identity-based secrets and encryption management — a single server that brokers, leases, and audits access to every credential in an infrastructure.

[GitHub repo](https://github.com/hashicorp/vault) ·
[Official website](https://developer.hashicorp.com/vault) ·
[License: BUSL-1.1](https://github.com/hashicorp/vault/blob/main/LICENSE)

## Overview

Vault is a Go server (and single CLI binary) for secrets management, encryption as a service, and privileged access management, first released by HashiCorp in 2015[^1]. Instead of scattering API keys, database passwords, and certificates across config files and environment variables, applications and operators ask Vault for them at runtime; Vault authenticates the caller, checks policy, hands back a credential with a lease, and records the access in an audit log. It is the reference implementation of the "dynamic secrets" pattern: rather than storing a long-lived database password, Vault holds admin credentials and mints short-lived, per-client credentials on demand, then revokes them when the lease expires.

The defining feature is that Vault is a *broker*, not just a store. Its `transit` engine encrypts and decrypts data without ever persisting it (encryption as a service); its `database`, `aws`, `pki`, and `kubernetes` engines generate credentials on the fly; its auth methods turn an existing identity (a Kubernetes service account, an AWS IAM role, an OIDC token) into a short-lived Vault token scoped by policy. This solves the "secret zero" problem partially — you still have to bootstrap trust to the first credential — but it collapses credential sprawl into one auditable choke point.

The defining tension is licensing and operational weight. In August 2023 HashiCorp relicensed Vault from MPL-2.0 to the Business Source License 1.1 (BUSL), which prohibits competing commercial use[^2]. This triggered the OpenBao fork under the Linux Foundation, which continues the MPL-2.0 lineage[^3]. The `NOASSERTION` license field GitHub reports reflects that BUSL is a source-available, not OSI-approved open-source, license. Separately, Vault is genuinely heavy to run correctly: it must be *unsealed* after every restart, its storage needs quorum and backups, and a misconfigured audit device can wedge the whole cluster.

## Getting Started

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
# or download a binary from https://releases.hashicorp.com/vault
```

```bash
# Dev server — in-memory, auto-unsealed, single node. NEVER for production.
vault server -dev
# copy the printed Root Token, then in another shell:
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='<root-token-from-output>'

# Static key/value secret (KV v2 engine, mounted at secret/ in dev)
vault kv put secret/myapp/config username=admin password=s3cr3t
vault kv get secret/myapp/config

# Encryption as a service — Vault never stores the plaintext
vault secrets enable transit
vault write transit/keys/my-key type=aes256-gcm96
vault write transit/encrypt/my-key plaintext=$(echo -n "card-number" | base64)
```

## Architecture / How It Works

Everything sits behind a **cryptographic barrier**: all data is encrypted with the barrier key before it touches the storage backend, so gaining raw access to the storage (a file, a Raft snapshot, a Consul KV) yields only ciphertext. The barrier key is itself encrypted by the **root key**, which is protected by the **unseal keys**. By default the unseal keys are Shamir shares — the root key is split into N shares with a threshold of K, and a quorum of key holders must provide K shares to *unseal* a freshly started (or restarted) Vault before it will serve any request.

- **Storage backend** — Vault is stateless compute over pluggable storage. **Integrated Storage (Raft)** is the current default and recommended backend; older deployments used Consul or other backends. Only one node is the active writer (the Raft leader); standbys forward writes.
- **Secrets engines** — mounted at paths (`kv/`, `database/`, `aws/`, `pki/`, `ssh/`, `transit/`). Static engines store; dynamic engines generate. Each dynamic secret carries a **lease** (TTL); at expiry Vault revokes it automatically.
- **Auth methods** — `token`, `approle`, `kubernetes`, `aws`, `ldap`, `jwt`/`oidc`, `github`, and more. An auth method verifies an external identity and issues a Vault **token** bound to one or more policies.
- **Policies** — HCL/JSON documents granting capabilities (`read`, `create`, `update`, `delete`, `list`) on paths. Deny by default; policies are additive.
- **Leases, renewal, revocation** — the expiration manager tracks every lease and can revoke a whole tree (e.g. all secrets issued to a user) in one call — the mechanism behind incident-response key rolling.
- **Audit devices** — every request/response is logged (with sensitive values HMAC'd) to file, syslog, or socket before the response is returned.

Enterprise adds namespaces, performance/DR replication, HSM-backed seals, and Sentinel policies — features that do not exist in the community binary.

## Production Notes

**The seal/unseal ceremony is the top operational footgun.** After any restart or crash, Vault comes up *sealed* and serves nothing until unsealed. Doing this by hand (gathering Shamir key holders) does not survive autoscaling or a 3am pod restart. Production deployments almost always configure **auto-unseal** delegating the root key to a cloud KMS, an HSM, or another Vault's transit engine. Plan this before go-live, not after the first outage.

**Audit devices fail closed.** If Vault cannot write to *any* enabled audit device, it will block requests rather than serve an unaudited one. A full disk or an unreachable syslog socket can therefore take down the whole cluster. Enable at least two audit devices of different types.

**Lease explosion.** High-churn dynamic secrets (short TTLs, many clients) generate huge numbers of leases the expiration manager must track and revoke; this has historically been a scaling and memory pressure point. **Batch tokens** (not persisted, no lease, cannot be renewed or revoked) exist specifically to relieve this load versus **service tokens** — but you trade away revocability. Choose deliberately.

**Storage is your durability story.** With Integrated Storage, losing Raft quorum makes Vault read-only or unavailable; take regular `vault operator raft snapshot` backups and test restores. Root/recovery keys and the initial root token must be stored out-of-band and ideally destroyed/regenerated after setup.

**Write throughput does not scale horizontally** in the community edition — one active node handles all writes. Read scaling via standbys helps reads only; write scaling across regions is a paid replication feature.

**KV v1 vs v2 confusion.** The versioned KV v2 engine wraps secrets under a `data/` path and adds metadata; CLI paths (`vault kv`) hide this but raw API paths do not. Mixing them up is a common first-week error.

**Upgrades and the license line.** Releases from Vault 1.15 (2023) onward are BUSL-licensed[^2]; 1.14.x and earlier remain MPL-2.0. Teams with a policy against source-available licenses either pin to pre-1.15 (unmaintained) or migrate to OpenBao. Vault-to-OpenBao migration is designed to be near drop-in for the 1.15-era fork point but diverges over time.

## When to Use / When Not

**Use when:**
- You need dynamic, short-lived credentials for databases, cloud IAM, or PKI issued and revoked on demand.
- You want one audited, policy-governed choke point for all secrets across many services and clouds (vendor-neutral, not tied to one cloud's KMS).
- You need encryption as a service so apps never hold key material.
- You require a full audit trail of who accessed which secret when.

**Avoid when:**
- You are single-cloud and a managed store (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) covers your needs with far less operational burden.
- You cannot staff the operational complexity — seal management, HA storage, backups, upgrades — for what is now critical-path infrastructure.
- Your organization's policy forbids source-available (non-OSI) licenses — evaluate OpenBao instead.
- You only need static secrets injected at deploy time; SOPS or sealed-secrets are lighter.

## Alternatives

- openbao/openbao — the Linux Foundation MPL-2.0 fork of Vault; use this when you want Vault's model and API but require an OSI open-source license.
- infisical/infisical — secrets management with a stronger developer/UI focus; use when team DX and simple setup matter more than dynamic-secret breadth.
- getsops/sops — file-level encryption for GitOps; use when you want secrets in version control with no server to run.
- external-secrets/external-secrets — Kubernetes operator syncing from cloud secret stores; use when you are all-in on Kubernetes and want native `Secret` objects.
- AWS Secrets Manager / Azure Key Vault / GCP Secret Manager — use when you are committed to one cloud and don't need vendor neutrality or Vault's dynamic engines.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-04 | Initial release[^1]. |
| 0.6 | 2016-06 | Response wrapping, official Consul storage. |
| 1.0 | 2018-11 | GA milestone; API stability commitments. |
| 1.4 | 2020-04 | Integrated Storage (Raft) reaches GA — Consul no longer required for HA. |
| 1.9 | 2021-11 | OIDC provider, expanded Kubernetes auth. |
| 1.15 | 2023-09 | First release under BUSL-1.1[^2]; UI and secrets-sync work. |
| 1.16 | 2024-03 | Secrets sync GA (Enterprise), event notifications. |

## References

[^1]: Mitchell Hashimoto, "Vault Announcement" — HashiCorp blog, 2015-04-28. https://www.hashicorp.com/blog/vault
[^2]: HashiCorp, "HashiCorp adopts Business Source License" — 2023-08-10. https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license
[^3]: OpenBao — Linux Foundation fork of Vault continuing the MPL-2.0 lineage. https://openbao.org
[^4]: Vault documentation — architecture, seal/unseal, secrets engines. https://developer.hashicorp.com/vault/docs

## Tags

go, secrets-management, encryption, pki, dynamic-secrets, identity, security, infrastructure, hashicorp, busl, devops
