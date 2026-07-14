# bentoml/BentoML

> A Python framework for packaging models and inference code into containerized HTTP services — with a commercial cloud attached.

[GitHub repo](https://github.com/bentoml/BentoML) ·
[Official website](https://bentoml.com) ·
[License: Apache-2.0](https://github.com/bentoml/BentoML/blob/main/LICENSE)

## Overview

BentoML is a model-serving framework: you write inference code as a Python class,
and BentoML turns it into an HTTP server, packages it with its models and
dependencies into a versioned artifact (a "Bento"), and builds an OCI image from
that artifact. It first appeared in 2019 as a way to wrap ML models in REST APIs;
the 1.0 release in mid-2022 was a ground-up rewrite that introduced the Bento
format and the Runner execution model[^1]. A second, equally disruptive rewrite
arrived with the 1.2 SDK in early 2024, replacing the Runner/Service split with a
single class-based `@bentoml.service` decorator[^2].

The project sits between "just write a FastAPI endpoint" and full inference
platforms like Triton or Ray Serve. Its value proposition is the packaging and
operational layer — dependency capture, reproducible builds, dynamic batching,
multi-worker parallelism, and a graph model for composing several models — rather
than a novel runtime. As of 2026 it is actively maintained (frequent commits,
~8.7k stars) and increasingly oriented toward LLM and generative workloads.

The defining tension is the same one that shapes many open-core infrastructure
projects: the OSS framework is genuinely usable standalone via Docker, but the
smoothest path (autoscaling, GPU scheduling, observability) runs through
BentoCloud, the company's commercial platform. The self-hosted Kubernetes story
(Yatai) exists but receives far less attention than BentoCloud.

## Getting Started

```bash
# Requires Python >= 3.9
pip install -U bentoml
```

```python
# service.py
import bentoml

@bentoml.service(
    image=bentoml.images.Image(python_version="3.11").python_packages(
        "torch", "transformers"
    ),
)
class Summarization:
    def __init__(self) -> None:
        from transformers import pipeline
        self.pipeline = pipeline("summarization")

    @bentoml.api(batchable=True)
    def summarize(self, texts: list[str]) -> list[str]:
        results = self.pipeline(texts)
        return [item["summary_text"] for item in results]
```

```bash
bentoml serve          # local dev server at http://localhost:3000
bentoml build          # package code + models + deps into a Bento
bentoml containerize summarization:latest   # generate an OCI image
```

## Architecture / How It Works

A **Service** is a Python class decorated with `@bentoml.service`. Public methods
tagged with `@bentoml.api` become HTTP endpoints; type hints on their signatures
drive request/response (de)serialization and generated OpenAPI schemas. Each
service runs as a pool of **worker** processes, which is how BentoML sidesteps the
GIL for CPU-bound inference and pins workers to specific GPUs.

A **Bento** is the deployable artifact: a directory (and tarball) bundling the
service code, a lockfile-style dependency manifest, saved model references, and
build configuration. `bentoml build` produces it and records it in a local Bento
Store; `bentoml containerize` renders a Dockerfile from the manifest and builds an
image via BuildKit. Models are handled separately through a local **Model Store**
(`bentoml.models.create` / `bentoml.models.get`), so weights can be versioned and
shared across services without living in the code repo.

Two features do most of the operational work:

- **Adaptive (dynamic) batching** — when an API is marked `batchable=True`, the
  server buffers concurrent requests and dispatches them as a batch, trading a few
  milliseconds of latency for throughput. Batch window and size are tunable.
- **Model composition / inference graphs** — a service can hold references to
  other services and call them, letting a request fan out across multiple models
  (ensembles, pre/post-processing stages) that each scale independently.

Historically, inference ran in separate **Runner** processes bridged to the API
server over an internal protocol. The 1.2 SDK collapsed that into services calling
services, which is simpler to reason about but means most pre-2024 tutorials and
examples describe an API that no longer exists[^2].

## Production Notes

**The 1.2 rewrite is a migration cliff.** Code written against the pre-1.2
Runner/`bentoml.Service(...)` API does not run on the current SDK; it needs a
rewrite, not a config bump. A large fraction of blog posts, Stack Overflow
answers, and even third-party course material still teach the old API. Verify that
any example you copy uses the `@bentoml.service` class form.

**Per-worker memory multiplies.** Each worker process is a full copy of the
service, so a model loaded in `__init__` is resident once per worker. With N
workers and a multi-GB model, host RAM (and GPU VRAM for GPU workers) is consumed
N times unless you explicitly reduce worker count or share weights. This is the
most common cause of OOM surprises when scaling up concurrency.

**Images get large.** Packaging PyTorch + CUDA + a model comfortably produces
multi-gigabyte images. Layer caching helps on rebuilds, but cold pulls in
autoscaling clusters are slow; plan registry bandwidth and node-local caching
accordingly.

**Self-hosting beyond Docker is the underinvested path.** `bentoml containerize`
plus your own orchestration works and is fully supported. But Kubernetes-native
deployment via Yatai is markedly less polished and less actively developed than
BentoCloud; teams wanting turnkey autoscaling and GPU scheduling on their own
infra should budget real integration work rather than expecting parity.

**Telemetry is opt-out.** BentoML reports anonymous usage of its internal API
calls by default. Disable with `--do-not-track` or `BENTOML_DO_NOT_TRACK=True` if
that matters in your environment[^3].

## When to Use / When Not

**Use when:**
- You have several models or a pre/post-processing pipeline and want one framework
  to package, containerize, and compose them.
- You want dynamic batching and multi-worker serving without hand-rolling it.
- You value reproducible, dependency-captured builds over a bare FastAPI endpoint.
- BentoCloud's managed autoscaling is an acceptable (or desired) deployment target.

**Avoid when:**
- You serve a single LLM and want maximum token throughput — vLLM or TGI directly
  is leaner (BentoML often wraps vLLM anyway).
- You need NVIDIA-grade GPU serving performance and standardized model formats —
  Triton is purpose-built for that.
- You already run everything on a Ray cluster — Ray Serve keeps you in one system.
- You just need one model behind one endpoint — plain FastAPI has less to learn.

## Alternatives

- vllm-project/vllm — use instead when the only job is high-throughput LLM token generation on GPU.
- ray-project/ray — use instead when serving lives inside a broader Ray distributed-compute workload.
- triton-inference-server/server — use instead when you need NVIDIA-optimized GPU serving and multi-framework model formats.
- pytorch/serve — use instead for a PyTorch-only shop wanting a lighter, framework-native server.
- kserve/kserve — use instead when you want Kubernetes-native, CRD-driven serving as the primitive.
- mlflow/mlflow — use instead when model registry and experiment tracking matter more than the serving layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019 | Initial releases: `BentoService` classes with `@api` and artifacts[^1]. |
| 1.0 | 2022-07 | Ground-up rewrite: the Bento format, Runners, Model Store[^1]. |
| 1.1 | 2023 | Incremental hardening of the 1.0 Runner architecture. |
| 1.2 | 2024-02 | New SDK: class-based `@bentoml.service`, Runner split removed[^2]. |
| 1.3+ | 2024–2026 | LLM/GenAI focus, image build API, ongoing BentoCloud integration. |

## References

[^1]: BentoML 1.0 announcement — the Bento standard and rewrite. https://www.bentoml.com/blog/announcing-bentoml-1-0
[^2]: BentoML 1.2 release — the new class-based service SDK. https://www.bentoml.com/blog/bentoml-1-2-simplify-model-serving
[^3]: Usage tracking and opt-out, BentoML README. https://github.com/bentoml/BentoML#usage-tracking-and-feedback

## Tags

python, model-serving, mlops, llm-serving, inference-api, machine-learning, docker, model-inference, open-core, generative-ai
