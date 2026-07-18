# google/go-cloud

> The Go Cloud Development Kit — vendor-neutral Go interfaces for blob storage, pub/sub, secrets, and config, so the same code runs on AWS, GCP, or Azure.

[GitHub repo](https://github.com/google/go-cloud) ·
[Official website](https://gocloud.dev/) ·
[License: Apache-2.0](https://github.com/google/go-cloud/blob/master/LICENSE)

## Overview

The Go Cloud Development Kit (Go CDK) is a set of Go libraries that abstract common cloud services behind provider-neutral interfaces. It was announced by Google in 2018 at Cloud Next '18[^1] with the tagline "write once, run on any cloud." The design analogy the maintainers use is `database/sql`: one interface (`blob.Bucket`, `pubsub.Topic`, `docstore.Collection`) with pluggable drivers (`s3blob`, `gcsblob`, `azureblob`, and so on), selected at runtime by URL scheme.

The defining tradeoff is **portability versus feature access**. The Go CDK deliberately exposes the intersection of what its target providers support, not the union. You get a stable API that moves between S3 and GCS with a one-line URL change, but you lose provider-specific features (S3 storage classes, GCS object holds, SNS message attributes beyond the common set) unless you reach through the `As` escape hatch to the underlying SDK type. For teams that genuinely deploy across clouds — or want to keep that option open without rewriting data-access code — the abstraction pays off. For teams committed to one provider, it is a layer of indirection over that provider's own SDK.

A second important fact: the APIs have been labeled "alpha" since launch, and the maintainers describe the project as being in a maintenance posture — actively supporting existing APIs and drivers, but explicitly unlikely to accept new ones into the core repository[^2]. Despite the alpha label, the packages are stable in practice and have seen little churn for years. Read "alpha" here as "we reserve the right to break, but rarely do," not "unstable."

## Getting Started

```shell
go get gocloud.dev
```

Reading from blob storage with a runtime-selected backend:

```go
package main

import (
	"context"
	"fmt"
	"io"

	"gocloud.dev/blob"
	_ "gocloud.dev/blob/s3blob"  // register the "s3://" scheme
)

func main() {
	ctx := context.Background()

	// Swap "s3://my-bucket" for "gs://..." or "azblob://..." with no code change.
	bucket, err := blob.OpenBucket(ctx, "s3://my-bucket?region=us-east-1")
	if err != nil {
		panic(err)
	}
	defer bucket.Close()

	r, err := bucket.NewReader(ctx, "my-blob", nil)
	if err != nil {
		panic(err)
	}
	defer r.Close()

	data, _ := io.ReadAll(r)
	fmt.Println(string(data))
}
```

The blank import (`_ "gocloud.dev/blob/s3blob"`) is load-bearing: it runs the driver's `init()` to register the URL scheme. Forgetting it produces a runtime "no driver registered for s3" error, not a compile error.

## Architecture / How It Works

The Go CDK is organized as one Go module (`gocloud.dev`) containing several **portable APIs**, each a package with a concrete type backed by a driver interface:

- `blob` — unstructured object storage (`s3blob`, `gcsblob`, `azureblob`, `fileblob`, `memblob`).
- `pubsub` — publish/subscribe messaging (AWS SNS/SQS, GCP Pub/Sub, Azure Service Bus, Kafka, NATS, RabbitMQ).
- `docstore` — document/NoSQL storage (DynamoDB, Firestore, MongoDB, Cosmos DB, in-memory).
- `runtimevar` — configuration values that change at runtime (AWS Parameter Store, GCP Runtime Configurator, etcd, file, constant).
- `secrets` — encryption/decryption via KMS-style key services (AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault, local).
- `mysql` / `postgres` — helpers for opening managed relational databases (Cloud SQL, RDS) with the right dialer and TLS.
- `server` — an opinionated HTTP server bundle with request logging, health checks, and tracing.

Two mechanisms make this work. The first is the **URL opener** pattern: `blob.OpenBucket(ctx, "gs://bucket")` parses the scheme, looks up the registered driver, and constructs the concrete type. This is why driver selection is a string, and why the blank-import registration matters. The second is the **`As` pattern**: when you need a provider-specific capability the portable API does not expose, you pass a pointer to the provider's SDK type into an `As(&target)` method, and the driver fills it in if the types match. This is the sanctioned escape hatch, and it is explicitly a portability-breaking one — code using `As` is coupled to a specific provider.

The project pairs with **google/wire**[^3], a compile-time dependency-injection code generator by the same team. Wire is not required, but it is the recommended way to assemble drivers so that a binary imports only the cloud SDKs it actually uses. Because each driver's SDK dependency is pulled in by import, naive use can balloon binary size and compile time; Wire (or careful manual import hygiene) keeps a multi-cloud-capable codebase from linking every provider's SDK into every build.

## Production Notes

**Binary size and build time.** Each driver transitively depends on its provider's full Go SDK (the AWS SDK for Go in particular is large). Importing `s3blob`, `gcsblob`, and `azureblob` together links three cloud SDKs. Import only the drivers a given binary needs, and lean on Wire so environment-specific builds don't drag in unused providers.

**The `As` escape hatch is a coupling decision, not a convenience.** Every call to `As` or a provider-specific URL parameter ties that code path to one backend. Audit `As` usage before claiming a service is "portable" — it is the most common place where a nominally multi-cloud codebase has quietly become single-cloud.

**Error handling loses provider detail by design.** The portable APIs surface a normalized error surface (`gcerrors.Code`, e.g. `NotFound`, `AlreadyExists`, `PermissionDenied`). Provider-specific error codes, retry hints, and throttling signals are flattened. If you rely on distinguishing a specific S3 or DynamoDB error condition, you will need `As` on the error or the underlying SDK.

**`pubsub` batching and ordering semantics vary by backend.** The portable interface hides provider differences, but delivery guarantees do not fully normalize — ordering, at-least-once vs at-most-once behavior, ack deadlines, and batch sizing differ between SNS/SQS, GCP Pub/Sub, and Kafka. Test against the real backend you deploy on; the in-memory driver will not reproduce these differences.

**"Alpha" in practice.** The APIs have carried the alpha label for the life of the project but are stable and widely used in production. The more relevant caution is velocity: the repository is in maintenance mode, new drivers are directed to external repositories[^2], and open-issue counts and commit activity are low. Treat it as a stable, slow-moving dependency — good for longevity, less good if you need an actively expanding driver ecosystem inside the core module.

**Naming collision.** The Go CDK is unrelated to "GoCloud Systems" at gocloud.systems; the canonical home is gocloud.dev. Do not conflate the two when searching for docs.

## When to Use / When Not

**Use when:**
- You deploy the same application across more than one cloud, or want to preserve that option without rewriting storage/messaging code.
- You want a `database/sql`-style seam over blob storage, pub/sub, secrets, or runtime config so backends are swappable by URL.
- You're writing Go and value a single, stable interface over learning each provider's SDK surface.
- You need local/in-memory drivers (`memblob`, `fileblob`, in-memory docstore) for tests that mirror the production interface.

**Avoid when:**
- You are committed to one cloud and want full access to that provider's feature set — use the native SDK directly.
- You need a service the Go CDK doesn't abstract (queues with exotic semantics, provider-specific analytics, streaming) — the abstraction adds indirection without payoff.
- You want an actively growing driver ecosystem maintained in-tree; the project's stated posture is maintenance, not expansion.
- You're not in Go; the Go CDK is Go-only and has no cross-language story.

## Alternatives

- aws/aws-sdk-go-v2 — use directly when you're all-in on AWS and want every service and feature without an abstraction layer.
- googleapis/google-cloud-go — the native GCP SDK; prefer it when single-cloud on Google and you need full GCP surface.
- Azure/azure-sdk-for-go — native Azure SDK for single-cloud Azure deployments.
- google/wire — companion, not competitor; the compile-time DI generator the Go CDK is designed to pair with for driver wiring.
- minio/minio-go — if your only portability need is S3-compatible object storage (MinIO, S3, R2, Wasabi), a single S3 client against compatible endpoints is simpler than a full abstraction layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-03 | Repository created[^4]. |
| Announcement | 2018-07 | Public launch at Cloud Next '18; `blob`, `pubsub`, `runtimevar`, `server`[^1]. |
| docstore | 2019 | Document storage API added (DynamoDB, Firestore, MongoDB, in-memory). |
| secrets | 2019 | KMS-style encryption API added. |
| Maintenance posture | ongoing | Maintainers direct new drivers to external repos; core APIs stable, still labeled alpha[^2]. |

## References

[^1]: "Building Go Applications for the Open Cloud" — Google Cloud Next '18, and the Go blog announcement. https://go.dev/blog/go-cloud
[^2]: Project status, README (google/go-cloud) — "we prefer to focus on maintaining the existing APIs and drivers, and are unlikely to accept new ones." https://github.com/google/go-cloud#project-status
[^3]: google/wire — compile-time dependency injection for Go, recommended for assembling Go CDK drivers. https://github.com/google/wire
[^4]: GitHub repository metadata, google/go-cloud (created 2018-03-21). https://github.com/google/go-cloud

## Tags

go, golang, cloud, multi-cloud, blob-storage, pubsub, abstraction-layer, aws, gcp, azure, portability
