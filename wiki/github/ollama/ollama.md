# ollama/ollama

> Run open-weight LLMs locally with a single command — a Go daemon that wraps model download, quantization, and an inference backend behind a REST API.

[GitHub repo](https://github.com/ollama/ollama) ·
[Official website](https://ollama.com) ·
[License: MIT](https://github.com/ollama/ollama/blob/main/LICENSE)

## Overview

Ollama is a local model runner: a background daemon (`ollama serve`) plus a CLI that pulls quantized open-weight models from a registry and serves them over an HTTP API on `localhost:11434`. It was first released in 2023 and is written in Go[^1]. Its value proposition is packaging — turning "download weights, pick a quantization, compile llama.cpp, wire up a chat template, manage a KV cache" into `ollama run gemma3`. The model catalog (`ollama.com/library`) and the `Modelfile` format (a Dockerfile-like recipe for weights + system prompt + parameters) are the two abstractions most users actually touch.

The audience is developers who want inference on their own hardware — laptops, workstations, a single GPU box — without sending prompts to a hosted API. That framing sets its defining tension: Ollama optimizes for single-machine ergonomics, not multi-tenant serving. It is deliberately not a datacenter inference server, and reaching for it as one (high concurrency, batched throughput, tensor parallelism across many GPUs) is where operators hit its limits. For those workloads, purpose-built servers like vLLM exist.

Historically Ollama was a thin, opinionated wrapper over [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)[^2]. Over time it has added its own Go-side inference engine built on GGML for newer (often multimodal) architectures, so the project now spans both a bundled llama.cpp runner and first-party model support — a coupling that shapes which models get day-one support and which lag.

## Getting Started

```shell
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh
# Windows: irm https://ollama.com/install.ps1 | iex
```

Pull and chat with a model from the terminal:

```shell
ollama run gemma3
```

The same daemon exposes a REST API — this is how applications integrate:

```shell
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3",
  "messages": [{ "role": "user", "content": "Why is the sky blue?" }],
  "stream": false
}'
```

Python (the official client talks to the local daemon over HTTP):

```python
from ollama import chat

response = chat(model="gemma3", messages=[
    {"role": "user", "content": "Why is the sky blue?"},
])
print(response.message.content)
```

Ollama also serves an OpenAI-compatible endpoint at `/v1/chat/completions`, which lets most OpenAI SDKs and tools point at it by changing the base URL.

## Architecture / How It Works

Ollama is a client/server split. The `ollama` CLI and every SDK are thin HTTP clients; all work happens in the daemon:

- **Model store & registry** — Models are content-addressed blobs plus a manifest, distributed over an OCI-style protocol from `registry.ollama.ai`. `ollama pull` fetches layers (weights, template, params, license); local models live under `~/.ollama/models`. A `Modelfile` (`FROM`, `SYSTEM`, `PARAMETER`, `TEMPLATE`, `ADAPTER`) builds a derived model on top of a base — the mechanism for baking in a system prompt, sampling params, or a LoRA adapter.
- **Weights format** — Models are **GGUF**, the quantized tensor format from the llama.cpp ecosystem[^2]. Default library tags are typically 4-bit quantizations (Q4_K_M-class), which is why a "7B" model is a few gigabytes rather than tens.
- **Inference backend** — For many models Ollama drives a bundled **llama.cpp** runner (GGML tensor library, CPU + CUDA/Metal/ROCm/Vulkan offload). For newer architectures it uses a first-party Go engine, also on GGML. Backend selection is per-model and largely invisible to the caller.
- **Scheduling & memory** — The daemon lazy-loads a model on first request, keeps it resident for a keep-alive window (default ~5 minutes), and evicts to free VRAM/RAM. It estimates how many layers fit in GPU memory and splits the rest to CPU. Concurrency (parallel requests, multiple loaded models) is bounded by env vars like `OLLAMA_NUM_PARALLEL` and `OLLAMA_MAX_LOADED_MODELS`.
- **Templating** — Each model ships a Go-templated prompt format so chat roles map to the exact tokens the model was trained on. Wrong or missing templates are a recurring source of "the model outputs garbage" reports for hand-imported GGUFs.

The co-evolution story is the crux: Ollama tracks llama.cpp for broad model coverage while building its own runner for models it wants to support first (and to control the multimodal and scheduling paths). New architectures are available roughly when one of those two backends supports them — Ollama does not implement transformer kernels from scratch.

## Production Notes

**It is a single-node runner, not a fleet server.** Throughput under concurrent load is the most common disappointment. Default settings favor one model, low parallelism, and interactive latency. For many simultaneous users or batched throughput, operators generally move to vLLM, TGI, or SGLang; using Ollama behind a busy service usually means fronting several daemons with a load balancer and accepting per-request (not batched) efficiency.

**Quantization defaults are a silent quality knob.** The convenient `ollama run <model>` pulls a 4-bit quant. That is fine for chat but visibly degrades on reasoning, code, and long-context tasks versus higher-precision tags (`:q8_0`, `:fp16`) — which most users never realize they can request. Pin an explicit tag when quality matters.

**Memory estimation is heuristic.** The layer-offload calculation can over- or under-commit VRAM, causing OOM crashes or unexpectedly slow CPU fallback. Mixed/multi-GPU setups, shared iGPU memory, and ROCm on consumer AMD cards are the rough edges. `OLLAMA_DEBUG=1` surfaces the offload decisions.

**Networking defaults are conservative for good reason.** The daemon binds to localhost. Exposing it (`OLLAMA_HOST=0.0.0.0`) puts an unauthenticated inference endpoint on the network — Ollama ships no auth, so anyone who can reach the port can run models and read/write your local model store. Put it behind a reverse proxy with auth if it must be remote. Also set `OLLAMA_ORIGINS` deliberately; browser-based tools have hit CORS restrictions here.

**Keep-alive and model swapping cost latency.** With several models and limited VRAM, requests that trigger an evict-and-reload pay multi-second cold loads. Tuning `keep_alive` per request and `OLLAMA_MAX_LOADED_MODELS` is the usual fix; pre-warming with an empty request is a common trick.

**Storage grows quietly.** Every tag is a full multi-GB blob; pulling variants accumulates fast under `~/.ollama`. There is no automatic GC — `ollama rm` is manual.

## When to Use / When Not

**Use when:**
- You want local, private inference on a laptop or single GPU box with minimal setup.
- You're prototyping against an OpenAI-compatible API without cloud cost or data egress.
- You need an easy way to try many open models and switch between them.
- You're embedding local inference in a desktop app or a low-concurrency internal tool.

**Avoid when:**
- You need high-concurrency or batched serving throughput — reach for vLLM/TGI/SGLang.
- You need maximum quality and are unwilling to manage quantization tradeoffs.
- You need multi-node/tensor-parallel serving of very large models.
- You require built-in authentication, multi-tenancy, or per-user quotas — Ollama has none.

## Alternatives

- [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) — the backend Ollama builds on; use it directly when you want full control over quantization, sampling, and build flags without the daemon abstraction.
- [vllm-project/vllm](https://github.com/vllm-project/vllm) — use instead for production serving: PagedAttention, continuous batching, high concurrent throughput on datacenter GPUs.
- [lmstudio](https://lmstudio.ai) — closed-source desktop GUI over llama.cpp; use when you want a graphical model browser and chat app rather than a CLI/daemon.
- [huggingface/text-generation-inference](https://github.com/huggingface/text-generation-inference) — use for Hugging Face-native, Rust-based hosted serving with tighter Transformers integration.
- [Mozilla-Ocho/llamafile](https://github.com/Mozilla-Ocho/llamafile) — use when you want a single self-contained executable bundling weights + runtime with zero install.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2023-06 | Repository created; macOS-first local runner over llama.cpp[^1]. |
| — | 2023 (H2) | Linux support; model library and `Modelfile` established. |
| — | 2024-02 | Windows preview; OpenAI-compatible `/v1` endpoint added[^3]. |
| — | 2024 | Tool/function calling and embeddings endpoints. |
| — | 2024-12 | Structured outputs (JSON-schema constrained generation). |
| — | 2025 | First-party Go/GGML engine for newer multimodal models alongside llama.cpp. |

(Ollama ships continuously rather than on prominent semantic-version milestones; dates above are month-level approximations except the repository creation date.)

## References

[^1]: ollama/ollama repository and release history. https://github.com/ollama/ollama/releases
[^2]: llama.cpp / GGML — the inference library and GGUF format Ollama builds on. https://github.com/ggml-org/llama.cpp
[^3]: Ollama OpenAI-compatibility documentation. https://docs.ollama.com/api

## Tags

llm, local-inference, go, llama-cpp, gguf, model-runner, rest-api, self-hosted, on-device-ai, quantization, developer-tools
