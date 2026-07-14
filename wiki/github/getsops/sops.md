# getsops/sops

> An editor of encrypted files that encrypts individual values in YAML, JSON, ENV, INI, and binary documents against KMS, age, and PGP keys.

[GitHub repo](https://github.com/getsops/sops) ·
[Official website](https://getsops.io) ·
[License: MPL-2.0](https://github.com/getsops/sops/blob/main/LICENSE)

## Overview

SOPS ("Secrets OPerationS") solves a narrow problem: how to keep secrets in a git repository without keeping them in cleartext. Unlike a vault server (Vault, Conjur) it has no daemon, no network service, and no central store — it is a single Go binary that encrypts and decrypts files in place. It was launched at Mozilla in 2015, donated to the CNCF as a Sandbox project in 2023, and is now maintained by a community group under the `getsops` org[^1].

The defining design choice is that SOPS encrypts **values, not whole files**. Given a YAML or JSON document it walks the tree and encrypts each leaf value while leaving keys, structure, and comments in cleartext. The result is a file that still diffs cleanly in git — you can see that `database.password` changed without seeing what it changed to. This is the feature that made SOPS the default secrets layer for GitOps toolchains (Flux, Argo CD, Helm, Kustomize, Terraform).

The tension is key management. SOPS does not hold your keys; it delegates trust to a **key backend** — a cloud KMS (AWS, GCP, Azure, HuaweiCloud), `age`, or PGP. Each file embeds a data key that is itself encrypted once per configured backend, so "who can decrypt this file" is entirely a function of who can call which KMS or who holds which private key. Get that wrong and you either lock yourself out or hand secrets to too many principals. SOPS is a good file format and a thin encryption engine; the hard part it hands back to you is IAM.

## Getting Started

```bash
# macOS
brew install sops
# Go
go install github.com/getsops/sops/v3/cmd/sops@latest
# or grab a release binary from GitHub
```

Encrypt with an `age` recipient (the simplest keyless-cloud path):

```bash
age-keygen -o key.txt           # writes a public/private keypair
export SOPS_AGE_KEY_FILE=key.txt

# encrypt only the values, in place
sops encrypt --age age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
  secrets.yaml > secrets.enc.yaml

# open in $EDITOR, re-encrypt on save
sops secrets.enc.yaml

# decrypt to stdout
sops decrypt secrets.enc.yaml
```

A `.sops.yaml` in the repo root binds file paths to key backends so contributors never pass `--age`/`--kms` by hand:

```yaml
creation_rules:
  - path_regex: prod/.*\.yaml$
    kms: arn:aws:kms:us-east-1:111122223333:key/abcd-...
  - path_regex: .*\.yaml$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

## Architecture / How It Works

SOPS uses **envelope encryption**. For each file it generates a random 256-bit **data key**, encrypts every leaf value with AES-256-GCM, then encrypts the data key once per configured backend and stores all those wrapped copies in a `sops` metadata block appended to the file.

- **Per-value encryption.** Each encrypted value becomes a string like `ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]`. The IV is derived per value, and the path to the value is mixed into the GCM additional-authenticated-data so values cannot be moved between keys. Keys, list structure, and comments stay in cleartext, which is what preserves readable diffs.
- **The `sops` metadata block.** Appended to the same file, it holds one entry per backend (`kms`, `gcp_kms`, `azure_kv`, `hc_vault`, `age`, `pgp`), each carrying the data key encrypted by that backend, plus a **MAC** over all encrypted values. The MAC is what detects tampering or a value silently deleted.
- **Decryption** is "try every backend until one succeeds." SOPS attempts each wrapped data key; the first one your environment can unwrap (an assumable AWS role, a loaded age key, a gpg-agent key) yields the data key, which then decrypts every value. This is why a file can be readable by CI (via KMS) and by a developer (via age) simultaneously.
- **Format awareness.** SOPS has parsers ("stores") for YAML, JSON, dotenv, INI, and an opaque binary mode. Binary mode base64-encrypts the whole file as a single value and is the escape hatch for formats it does not understand (certs, arbitrary blobs).
- **Key rotation** (`sops rotate`) generates a fresh data key and re-wraps it, without you seeing the plaintext key. `sops updatekeys` re-encrypts the data key for a changed set of recipients after you edit `.sops.yaml`.

The Go module path is `github.com/getsops/sops/v3`; the library is importable, and much of the ecosystem (helm-secrets, the Flux `SOPS` decryptor, `terraform-provider-sops`) calls it as a library rather than shelling out.

## Production Notes

- **The file format is the API.** The encrypted-value string format and the `sops` metadata schema are extremely stable — files encrypted years ago still decrypt. Treat the format as a long-term commitment; downstream tools (Flux, Argo) parse it directly.
- **KMS is a hard runtime dependency at decrypt time.** With cloud KMS backends, every decrypt is an API call. This couples your app boot / CI pipeline to KMS availability, IAM correctness, and (at scale) KMS request quotas and per-request cost. `age` and PGP decrypt fully offline, which is why many teams prefer age for local/dev and reserve KMS for CI/prod.
- **MAC-mismatch on partial edits.** If a tool rewrites part of a SOPS file without going through SOPS (a naive YAML formatter, a merge that reorders keys), the MAC no longer matches and decryption fails with `MAC mismatch`. Always edit through `sops`, and be wary of pre-commit hooks that reformat YAML.
- **`sops updatekeys` is not automatic.** Adding a recipient to `.sops.yaml` does **not** re-encrypt existing files — new creation rules only apply to newly created files. You must run `sops updatekeys` across existing files, which is easy to forget when onboarding a new team member or rotating an offboarded one's key out.
- **Secrets still hit disk and process memory decrypted.** SOPS solves at-rest-in-git; it does not sandbox runtime. `sops exec-env` / `exec-file` inject secrets into a child process, but decrypted values live in that process's environment/argv where other tooling may capture them.
- **PGP is the legacy footgun.** GPG-agent state, expired subkeys, and `gpg` version differences cause the majority of "works on my machine" decrypt failures. New projects generally choose `age` (single small keypair, no agent) over PGP.
- **Not a secret scanner or access log.** SOPS records which keys *can* decrypt a file but not who *did*. Audit trails come from the underlying KMS (CloudTrail etc.), not from SOPS.

## When to Use / When Not

**Use when:**
- You practice GitOps and want secrets versioned alongside manifests with readable diffs.
- You want a serverless, single-binary secrets story with no infrastructure to run.
- You need one file decryptable by several trust domains at once (CI via KMS, humans via age).
- You already have a cloud KMS and want git as the source of truth.

**Avoid when:**
- You need dynamic secrets, leases, or short-TTL credentials — that is HashiCorp Vault's domain, not SOPS's.
- You need centralized runtime access auditing and revocation of individual reads.
- Your secrets change many times per minute or must never touch a git history.
- You cannot tolerate a KMS API call on the critical path and cannot use offline age/PGP.

## Alternatives

- hashicorp/vault — use instead when you need a running secrets server with dynamic secrets, leasing, and fine-grained runtime auditing rather than files in git.
- mozilla/sops — the pre-donation repository name; now redirects here. Historical references and old module paths point at it.
- FiloSottile/age — use directly when you only need to encrypt whole files/streams and do not need per-value encryption or multi-backend metadata.
- bitnami-labs/sealed-secrets — use instead when you are Kubernetes-only and want the cluster controller (not a KMS) to be the sole decryptor of Secrets.
- dani-garcia/vaultwarden or 1Password/doppler — use instead when you want a hosted secrets manager and UI rather than a git-native file format.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015 | Launched as a Mozilla project[^1]. |
| 3.0 | 2018 | Major release line; multi-backend envelope encryption, `.sops.yaml` creation rules. |
| 3.7 | 2021 | `age` backend support added, reducing reliance on PGP[^2]. |
| — | 2023 | Donated to the CNCF as a Sandbox project; repo moved to `getsops/sops`, module path `getsops/sops/v3`[^1]. |
| 3.8 | 2023 | First release under the new maintainer group after the org move. |

## References

[^1]: SOPS README — project history, CNCF Sandbox donation (2023), and maintainer stewardship. https://github.com/getsops/sops
[^2]: `age` — modern file encryption tool used as a SOPS key backend. https://github.com/FiloSottile/age

## Tags

go, secrets-management, encryption, devops, gitops, kms, age, pgp, cli, security, cncf, configuration
