# huggingface/text-generation-inference

> Hugging Face's production LLM serving stack — a Rust router in front of Python model shards — now in maintenance mode as the ecosystem consolidates on vLLM and SGLang.

[GitHub repo](https://github.com/huggingface/text-generation-inference) ·
[Official docs](https://huggingface.co/docs/text-generation-inference) ·
[License: Apache-2.0](https://github.com/huggingface/text-generation-inference/blob/main/LICENSE)

## Overview

Text Generation Inference (TGI) is a toolkit for deploying and serving large language models over HTTP/gRPC, first released in late 2022 to power Hugging Face's own production surfaces: HuggingChat, the Inference API, and Inference Endpoints[^1]. Architecturally it pairs a Rust router/launcher (request scheduling, continuous batching, SSE streaming) with Python "shard" processes that run the actual model with Flash Attention and Paged Attention kernels. It was one of the first widely deployed servers to ship continuous batching and paged KV-cache outside of vLLM.

As of 2026 the project is officially **in maintenance mode and the GitHub repo is archived**[^2]. The maintainers now recommend vllm-project/vllm and sgl-project/sglang for new deployments, and describe TGI as having "initiated the movement" toward inference engines built on `transformers` model architectures — a direction those downstream engines adopted[^2]. This is the defining fact about TGI today: it is stable, battle-tested, and no longer where feature work happens. New PRs are limited to minor bug fixes, docs, and lightweight maintenance.

The defining historical tension was licensing. In mid-2023 Hugging Face relicensed TGI under a custom source-available license (HFOIL) that restricted commercial hosted use, then reverted to Apache-2.0 in early 2024 after community backlash[^3]. The repo today is Apache-2.0, but the episode pushed some users toward vLLM during the window and is worth knowing before you build a product on any HF-controlled serving layer.

## Getting Started

The supported path is the official Docker container; local builds require Rust, Protoc, and CUDA kernels.

```shell
model=HuggingFaceH4/zephyr-7b-beta
volume=$PWD/data   # persist weights across runs

docker run --gpus all --shm-size 1g -p 8080:80 -v $volume:/data \
    ghcr.io/huggingface/text-generation-inference:3.3.5 --model-id $model
```

```bash
# OpenAI-compatible Messages API
curl localhost:8080/v1/chat/completions -X POST \
  -H 'Content-Type: application/json' \
  -d '{"model":"tgi","messages":[{"role":"user","content":"What is deep learning?"}],"stream":true,"max_tokens":64}'
```

The `--shm-size 1g` flag is not optional for multi-GPU tensor parallelism: NCCL falls back to host shared memory when peer-to-peer NVLink/PCIe is unavailable, and an under-sized `/dev/shm` produces cryptic NCCL hangs.

## Architecture / How It Works

TGI is three processes with a clear split of responsibilities:

1. **Launcher** (Rust) — parses CLI/env config, downloads weights (Safetensors), spawns one shard per GPU, and supervises them.
2. **Router** (Rust) — the HTTP/gRPC server. Owns the request queue, **continuous batching** (new requests join an in-flight batch between decode steps instead of waiting for the batch to drain), token streaming over Server-Sent Events, and backpressure. This is the part that makes throughput scale.
3. **Server / shards** (Python) — load the model and run forward passes. Tensor Parallelism splits weights across GPUs; Flash Attention and Paged Attention (paged KV cache, the vLLM technique) are the core kernels. The router talks to shards over gRPC (`proto/v3/generate.proto`).

Continuous batching plus paged attention is the throughput story: KV-cache memory is allocated in fixed blocks rather than per-request contiguous buffers, so many concurrent sequences share VRAM without over-allocation, and the router keeps the GPU saturated. Speculation (draft-model / n-gram) offers roughly 2x latency reduction on suitable workloads[^1], and guided/JSON generation constrains decoding to a grammar.

Quantization is broad: bitsandbytes (incl. NF4/FP4 4-bit), GPT-Q, AWQ, EETQ, Marlin, and fp8. "Optimized" model architectures (Llama, Mistral, Falcon, StarCoder, BLOOM, GPT-NeoX, and many more) get hand-written fast paths; anything else falls back to a generic `AutoModelForCausalLM` path with lower performance.

## Production Notes

- **Maintenance mode is the headline operator fact.** TGI works and is used in production, but you are adopting a frozen target. Expect no new model-architecture support beyond what shipped, and no new hardware backends. For anything cutting-edge (newest models, latest attention/quant work), vLLM and SGLang move faster.
- **Shared memory footgun.** Multi-GPU deployments hang silently without `--shm-size 1g` (Docker) or an `emptyDir{medium: Memory}` mount at `/dev/shm` (Kubernetes). `NCCL_SHM_DISABLE=1` avoids it at a performance cost.
- **CPU is not a target.** Removing `--gpus all` and adding `--disable-custom-kernels` runs, but the maintainers explicitly call CPU performance subpar[^1]. This is a GPU serving engine.
- **Gated models need `HF_TOKEN`.** Serving Llama and other gated weights requires a READ token passed as an env var; weight download happens at container start, so cold starts on large models are dominated by download + load time unless the `/data` volume is warm.
- **Hardware backends are separate images/repos.** AMD ROCm uses a `-rocm` tag; Gaudi, Inferentia/Neuron, TPU, and Intel GPU live in adjacent repos (`tgi-gaudi`, `optimum-neuron`, `optimum-tpu`) with their own version cadences and gaps. The Nvidia path is the first-class one.
- **Observability is built in.** OpenTelemetry distributed tracing (`--otlp-endpoint`) and Prometheus metrics ship in the box — a genuine operational advantage over rolling your own around a bare model server.
- **Licensing history.** Apache-2.0 today, but the 2023 HFOIL detour[^3] means the license terms are a Hugging Face policy decision, not a community-governed constant.

## When to Use / When Not

**Use when:**
- You already run on Hugging Face Inference Endpoints or want the exact stack HF operates.
- You need a stable, well-instrumented server for a fixed set of already-supported models and value maturity over new features.
- You want built-in OpenTelemetry/Prometheus and an OpenAI-compatible Messages API without assembly.

**Avoid when:**
- You need day-one support for the newest models or quantization schemes — that work now lands in vLLM/SGLang, not TGI.
- You're starting a greenfield deployment in 2026 — the maintainers themselves point you to vLLM or SGLang[^2].
- You want a community-governed project immune to a single vendor's licensing decisions.

## Alternatives

- vllm-project/vllm — the reference PagedAttention engine and the maintainers' recommended successor; use it as the default for new self-hosted LLM serving.
- sgl-project/sglang — high-throughput serving with RadixAttention and strong structured-output/agent workloads; use when prefix caching and complex generation control matter.
- NVIDIA/TensorRT-LLM — use when you're all-Nvidia and want maximum single-hardware performance and are willing to pay in build/compile complexity.
- ollama/ollama — use for local/desktop and small-team serving where ease beats raw multi-GPU throughput.
- ggml-org/llama.cpp — use for CPU, edge, quantized GGUF, and non-CUDA hardware where a Python/CUDA stack is impractical.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2022-10 | Rust router + Python shards; continuous batching, TP, SSE streaming[^1]. |
| HFOIL relicense | 2023-07 | Switched to source-available HFOIL restricting commercial hosted use[^3]. |
| Apache-2.0 restored | 2024-04 | Reverted to Apache-2.0 after community pushback[^3]. |
| 2.x line | 2024 | Broad quant support (AWQ/GPT-Q/Marlin/fp8), guided JSON, speculation. |
| 3.x line | 2025 | `transformers`-backend architecture direction; multi-hardware images. |
| Maintenance mode | 2026 | Repo archived; bug-fix-only; points users to vLLM/SGLang[^2]. |

## References

[^1]: TGI README and feature list — huggingface/text-generation-inference. https://github.com/huggingface/text-generation-inference
[^2]: TGI README maintenance-mode notice — recommends vllm and SGLang going forward. https://github.com/huggingface/text-generation-inference#readme
[^3]: Hugging Face relicensing of TGI to HFOIL (2023) and reversion to Apache-2.0 (2024). https://github.com/huggingface/text-generation-inference/blob/main/LICENSE
[^4]: "LLM inference at scale with TGI" — Martin Iglesias Goyanes, Adyen, 2024. https://www.adyen.com/knowledge-hub/llm-inference-at-scale-with-tgi

## Tags

python, rust, llm-inference, model-serving, gpu, continuous-batching, paged-attention, quantization, huggingface, maintenance-mode
