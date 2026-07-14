# boto/boto3

> The official AWS SDK for Python — a thin, data-driven wrapper over botocore that mirrors every AWS service API.

[GitHub repo](https://github.com/boto/boto3) ·
[Official website](https://aws.amazon.com/sdk-for-python/) ·
[License: Apache-2.0](https://github.com/boto/boto3/blob/develop/LICENSE)

## Overview

Boto3 is the Amazon Web Services SDK for Python, maintained and published by AWS itself. It lets Python code call S3, EC2, DynamoDB, Lambda, and the roughly 300+ other AWS services through a uniform interface. It is one of the most-installed packages on PyPI — effectively unavoidable for anyone writing Python against AWS — and is pre-bundled into the AWS Lambda Python runtime. The name is a reference to the Amazon river dolphin, chosen by Mitch Garnaat, author of the original `boto` library[^1].

The defining fact about boto3 is that it is *generated, not hand-written*. Almost nothing about a specific service lives in the boto3 codebase; instead, the underlying `botocore` library ships JSON models describing each service's operations, shapes, and errors, and the client for a service is built at runtime from those models[^2]. This is why boto3 can support new AWS services and API changes almost as fast as AWS ships them, and why a `boto3` upgrade is often the only thing needed to reach a new region or operation. It is also why the API is dynamically typed to the point of near-opacity: every operation takes and returns untyped `dict`s, method signatures are `**kwargs`, and your editor cannot autocomplete a `put_object` call without external help.

There are two API layers, and the split matters. Low-level **clients** (`boto3.client('s3')`) map one-to-one onto the service's API operations. Higher-level **resources** (`boto3.resource('s3')`) offer an object-oriented facade (`bucket.objects.all()`). The resource interface is in maintenance mode — it does not cover all services, receives no new features, and AWS has signaled it will not carry forward into a future major version[^3]. New code should prefer clients.

## Getting Started

```bash
python -m pip install boto3
```

```python
import boto3

# Low-level client — 1:1 with the S3 API
s3 = boto3.client("s3")

# Reuse a paginator for any truncating list operation
paginator = s3.get_paginator("list_objects_v2")
for page in paginator.paginate(Bucket="my-bucket", Prefix="logs/"):
    for obj in page.get("Contents", []):
        print(obj["Key"], obj["Size"])
```

Credentials are resolved from a chain: explicit args, environment variables (`AWS_ACCESS_KEY_ID` …), the shared `~/.aws/credentials` file, then IAM role metadata (EC2/ECS/Lambda). You almost never pass keys in code — on AWS compute, the instance/role identity is picked up automatically.

## Architecture / How It Works

Boto3 sits on top of **botocore**, which is the real engine and is shared with the AWS CLI[^2]. The layering is:

- **botocore** — loads the JSON service models, builds request signatures (SigV4), serializes/deserializes wire formats (REST-JSON, REST-XML, query, JSON-RPC), and handles the credential chain, endpoint resolution, and retries. All of this is data-driven from the bundled model files.
- **boto3** — adds the `Session` object, the `resource()` abstraction, collections, waiters exposed ergonomically, and a handful of high-level customizations (notably the S3 `transfer` manager for multipart upload/download).

Because clients are assembled from JSON at runtime, `boto3.client("service")` does real work the first time: it parses model files and constructs the client class dynamically. This has three consequences that surface constantly in production. First, client creation is comparatively expensive, so clients should be created once and reused. Second, there is no static type information — hence the ecosystem of `boto3-stubs` / `mypy_boto3_*` stub packages generated from the same models[^4]. Third, service data is *bundled* with the installed version, so a stale boto3 install silently lacks newer regions, operations, or parameters until you upgrade.

Thread-safety follows the client/resource split: a `Session` is not thread-safe, and neither are resource objects, but a low-level client is generally safe to share across threads once created. The common pattern in multithreaded code is to create one client and hand it to a thread pool[^5].

Pagination and waiters are first-class. Many list operations truncate at 1,000 items and return a continuation token; the `get_paginator` helper hides the token loop. Waiters (`client.get_waiter("bucket_exists")`) poll until a resource reaches a state, with model-defined intervals.

## Production Notes

**Reuse clients; create them outside the hot path.** In AWS Lambda, instantiate clients at module scope, not inside the handler, so warm invocations skip the model-parsing cost. Recreating a client per request is a common, measurable latency and CPU tax.

**Pin your own boto3 in Lambda.** The Lambda runtime bundles a boto3 version that is frequently *older* than the current release and changes without your control. If you depend on a recently added parameter or service, package your own version in the deployment artifact or a layer rather than relying on the runtime's copy.

**Tune retries and timeouts explicitly.** Defaults use the `legacy`/`standard` retry mode with modest attempt counts; for high-throughput or throttling-prone workloads, pass a `botocore.config.Config(retries={"mode": "adaptive", "max_attempts": N}, connect_timeout=…, read_timeout=…)`. The default connection pool is small (10), which silently serializes concurrent calls under load — raise `max_pool_connections` for parallel work.

**Types are bolt-on.** Untyped `dict` returns make refactors dangerous. Install `boto3-stubs[s3,dynamodb,…]` (from `mypy_boto3_builder`) to get real autocomplete and mypy coverage[^4]. Without it, `response["Item"]["attr"]["S"]` typos are runtime failures.

**Pagination is not optional.** Calling `list_objects_v2` or `scan` once and trusting the result is a classic bug — you get the first page and silently miss the rest. Always use paginators for list/scan/describe operations.

**Session vs. default session.** The module-level `boto3.client(...)` uses an implicit default session tied to the default credential/region resolution. For multi-account, multi-region, or assumed-role code, create explicit `boto3.Session(...)` objects rather than mutating global state.

**Python version churn.** AWS tracks the Python Software Foundation's support lifecycle; support for Python 3.9 ended for boto3 on 2026-04-29, following the runtime's own end of support on 2025-10-31[^6]. Plan interpreter upgrades accordingly.

## When to Use / When Not

**Use when:**
- You are writing Python against AWS at all — this is the official, canonical SDK.
- You need broad, current service coverage and fast access to new AWS features.
- You want IAM-role credential handling and request signing done for you.

**Avoid / look elsewhere when:**
- You need native asyncio — boto3 is synchronous and blocking; use an async wrapper instead.
- You want a strongly-typed, ergonomic API out of the box — the untyped dict interface is a real ongoing cost without stubs.
- You only need one or two operations in a size-constrained environment — the full SDK plus bundled models is a large dependency.

## Alternatives

- boto/botocore — the low-level core boto3 is built on; use it directly when you want signing/serialization without the resource layer and are building your own abstraction.
- aio-libs/aiobotocore — asyncio-native AWS calls over botocore; use it when boto3's blocking I/O is the bottleneck.
- aws/aws-cli — the command-line tool over the same botocore core; use it for shell scripting and ad-hoc operations instead of Python.
- getmoto/moto — in-process mocking of AWS APIs; use it to test boto3 code without hitting real AWS.
- youtype/mypy_boto3_builder — generates `boto3-stubs`; use it alongside boto3 to add static types and autocomplete.

## History

| Version | Date | Notes |
|---------|------|-------|
| boto (2.x) | pre-2015 | Original hand-written library by Mitch Garnaat; per-service code maintained by hand[^1]. |
| boto3 preview | 2014-10 | Repository created; data-driven rewrite over botocore begins[^2]. |
| boto3 1.0 (GA) | 2015-06-22 | General availability; full-support phase of the SDK lifecycle[^6]. |
| 1.x (rolling) | 2015–present | Continuous `1.x` releases; version tracks bundled botocore/service models rather than API-breaking majors. |
| — | 2026-04-29 | Python 3.9 support dropped, following PSF end of support[^6]. |

## References

[^1]: Boto3 README — naming and origin (Mitch Garnaat, Amazon river dolphin). https://github.com/boto/boto3/blob/develop/README.rst
[^2]: botocore — the low-level, data-driven core shared by boto3 and the AWS CLI. https://github.com/boto/botocore
[^3]: Boto3 docs, "Resources" — maintenance mode and feature freeze of the resource interface. https://boto3.amazonaws.com/v1/documentation/api/latest/guide/resources.html
[^4]: mypy_boto3_builder / boto3-stubs — generated type stubs for boto3. https://github.com/youtype/mypy_boto3_builder
[^5]: Boto3 docs, "Concurrency / multithreading" — client vs. session/resource thread-safety. https://boto3.amazonaws.com/v1/documentation/api/latest/guide/clients.html
[^6]: Boto3 README, "Notices" and "Maintenance and Support" — GA date 2015-06-22 and Python 3.9 end of support 2026-04-29. https://github.com/boto/boto3/blob/develop/README.rst

## Tags

python, aws, aws-sdk, cloud, sdk, botocore, s3, boto3, api-client, infrastructure
