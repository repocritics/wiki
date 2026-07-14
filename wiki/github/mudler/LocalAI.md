# mudler/LocalAI

> A self-hosted, OpenAI-compatible inference server that fronts many model backends (llama.cpp, vLLM, whisper.cpp, diffusers, MLX...) behind one API.

[GitHub repo](https://github.com/mudler/LocalAI) ·
[Official website](https://localai.io) ·
[License: MIT](https://github.com/mudler/LocalAI/blob/master/LICENSE)

## Overview

LocalAI is a Go server that exposes OpenAI-shaped REST endpoints (`/v1/chat/completions`, `/v1/embeddings`, `/v1/audio`, `/v1/images`...) and routes each request to a pluggable backend that actually runs the model. It was started by Ettore Di Giacinto (`mudler`) in early 2023 as a drop-in, self-hosted replacement for the OpenAI API, originally under the `go-skynet` org[^1]. Since then its scope has widened well past text: it now covers vision, speech-to-text, text-to-speech, image and video generation, reranking, object detection, and a built-in agent runtime — all behind the same API surface, and additionally speaking the Anthropic and ElevenLabs API dialects.

The defining design choice is that **the core is small and the backends are separate**. Each backend (the code that links llama.cpp, vLLM, stable-diffusion, whisper.cpp, MLX, and so on) ships as its own OCI image and is pulled on demand the first time a model needs it, rather than being compiled into one monolithic binary. This happened in the v3.2.0 rework (July 2025) that moved every backend out of the main binary[^2]. The upside is a lean install and per-model isolation; the cost is that LocalAI's real behavior — speed, quantization support, hardware acceleration — is inherited from whichever upstream engine a given model uses, and LocalAI is mostly orchestration, config templating, and API translation on top.

The project's ambition is breadth: "any model, any modality, any hardware, no GPU required." That breadth is genuine but is also the main source of friction — the surface area is large, the release cadence is fast, and a lot of the complexity moves from "which model" to "which backend image and which hardware flavor."

## Getting Started

```bash
# CPU-only, via container
docker run -ti --name local-ai -p 8080:8080 localai/localai:latest

# NVIDIA GPU (CUDA 12) — note the hardware-specific image tag
docker run -ti --name local-ai -p 8080:8080 --gpus all \
  localai/localai:latest-gpu-nvidia-cuda-12
```

Load and run a model from the gallery, then hit it with a stock OpenAI client:

```bash
# Pull + serve a model (gallery name, Hugging Face, Ollama, or OCI ref all work)
local-ai run llama-3.2-1b-instruct:q4_k_m
```

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.2-1b-instruct:q4_k_m",
    "messages": [{"role": "user", "content": "Explain quantization in one sentence."}]
  }'
```

Because the API is OpenAI-compatible, existing SDKs work by pointing `base_url` at the LocalAI host and using any non-empty API key.

## Architecture / How It Works

LocalAI is a Go HTTP/gRPC server. The request path is roughly:

1. **API layer** — translates an incoming OpenAI/Anthropic/ElevenLabs request into an internal call.
2. **Model config** — each model is described by a YAML file (in the model directory or pulled from the gallery) that pins the backend, the prompt/chat template, context size, and generation defaults.
3. **Backend process** — LocalAI starts the matching backend as a separate process and talks to it over gRPC. Backends implement a common interface, so a new engine can be added in any language against that contract.
4. **Backend gallery** — backends are distributed as OCI images and installed on the fly; the server downloads the one a model needs the first time it is used[^2].

Models can be sourced from the built-in gallery, Hugging Face (`huggingface://`), the Ollama registry (`ollama://`), a YAML URL, or any OCI registry (`oci://`). Beyond single-node serving, LocalAI has a **distributed mode** for horizontal scaling (a request router with VRAM/prefix-aware routing, backed by PostgreSQL + NATS) and a libp2p-based **P2P mode** for federating inference across machines[^3].

Several of the newer speech/vision backends are native C/C++/GGML engines written and maintained by the project itself (parakeet.cpp for ASR, vibevoice.cpp for TTS, rf-detr.cpp / locate-anything.cpp for detection, face-detect.cpp, etc.) rather than thin wrappers around Python libraries — a deliberate move to drop Python and onnxruntime from the inference path for those modalities[^4].

The critical coupling to understand: **the model YAML is where correctness lives.** The chat template, stop tokens, and backend selection in that file determine whether output is coherent. A model that produces garbage under LocalAI is usually a template/config mismatch, not a broken engine.

## Production Notes

**Hardware image selection is a real footgun.** GPU acceleration requires the *correct* image variant — `-gpu-nvidia-cuda-12`, `-gpu-nvidia-cuda-13`, `-gpu-hipblas` (AMD ROCm), `-gpu-intel` (oneAPI), `-gpu-vulkan`, or an L4T/Jetson tag. Run the plain `latest` image on a GPU box and you silently get CPU inference. Match the CUDA major version to your driver.

**"No GPU required" means it runs, not that it's fast.** CPU inference works for small quantized models; throughput and latency are bounded by the underlying backend (llama.cpp et al.), and large models on CPU are slow. LocalAI does not make the engine faster — it orchestrates it.

**Backends are pulled on demand.** First use of a new modality downloads an OCI image, so the first request can be slow and needs registry access and disk headroom. In air-gapped or bandwidth-constrained deployments, pre-pull the backend images you need. Container images across the acceleration matrix are large.

**Fast release cadence, moving surface.** The project ships major features monthly and has had structural changes across versions (backends leaving the binary in v3.2.0, distributed mode maturing through v4.x). Pin an image tag rather than tracking `latest` in production, and read release notes before upgrading — UI, config, and backend packaging have all shifted between minor versions[^5].

**macOS DMG is unsigned.** The desktop build is not Apple-signed; after install you must clear the quarantine attribute (`sudo xattr -d com.apple.quarantine /Applications/LocalAI.app`) or Gatekeeper blocks it[^6].

**Multi-user / auth is opt-in.** API-key auth, per-user quotas, and OIDC exist (added through the v4.1/v4.2 line) but are not the default; a bare server is unauthenticated. Put it behind auth before exposing it.

## When to Use / When Not

**Use when:**
- You want one self-hosted, OpenAI-compatible endpoint spanning text, vision, audio, and image/video without wiring up separate servers per modality.
- Data residency/privacy requires inference stay on your infrastructure.
- You want to swap engines (llama.cpp ↔ vLLM ↔ MLX) behind a stable API without changing client code.
- You're on heterogeneous hardware (mixed NVIDIA/AMD/Intel/Apple) and want one project that covers all of it.

**Avoid when:**
- You only need a single LLM locally with minimal ceremony — a narrower runner is less to operate.
- You need maximum GPU serving throughput for one model in production — a dedicated inference server will beat the general-purpose orchestrator.
- You want a small, slow-moving dependency — the surface area and release velocity are large.
- You need the underlying engine's newest flags/features the day they land — LocalAI trails the upstream backend it wraps.

## Alternatives

- ollama/ollama — simpler single-focus local LLM runner with the smoothest UX; use it when you want text (and some vision) models running with minimal configuration and don't need the multimodal breadth.
- vllm-project/vllm — high-throughput, GPU-first LLM serving engine; use it when you're serving one or few LLMs to many concurrent users and care about tokens/sec over modality coverage.
- ggml-org/llama.cpp — the inference engine LocalAI most often wraps; use it directly when you only need local LLM inference and don't want an API/orchestration layer.
- huggingface/text-generation-inference — production LLM serving tied to the Hugging Face ecosystem; use it when you're standardized on HF tooling and models.
- oobabooga/text-generation-webui — multi-backend local UI aimed at experimentation; use it when interactive tinkering matters more than a stable API.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-03 | First release: self-hosted OpenAI-compatible inference API (`go-skynet/LocalAI`)[^1]. |
| v3.2.0 | 2025-07 | Backends moved out of the main binary — modular, on-demand backend architecture[^2]. |
| v3.10.0 | 2026-01 | Anthropic API support, Open Responses API, LTX-2 video, unified GPU backends[^5]. |
| v4.0.0 | 2026-03 | Agentic orchestration + Agenthub, React UI rewrite, MCP apps, WebRTC realtime audio. |
| v4.1.0 | 2026-04 | Distributed cluster mode with VRAM-aware routing, multi-user platform (OIDC/quotas), in-UI fine-tuning. |
| v4.2.0 | 2026-05 | Voice + face recognition, drop-in Ollama API, video generation, vLLM at parity with llama.cpp. |
| v4.3.0 | 2026-05 | llama.cpp prompt cache on by default, cosign-signed backend images, Distributed v3 routing. |

## References

[^1]: Ettore Di Giacinto (`mudler`), LocalAI project — original author and lead. https://github.com/mudler/LocalAI
[^2]: LocalAI v3.2.0 release — backends migrated outside the main binary. https://github.com/mudler/LocalAI/releases/tag/v3.2.0
[^3]: LocalAI docs, Distributed Mode (PostgreSQL + NATS) and P2P inferencing. https://localai.io/features/distributed-mode/
[^4]: LocalAI README, "Backends built by us" — native C/C++/GGML engines. https://github.com/mudler/LocalAI#backends-built-by-us
[^5]: LocalAI releases and news. https://github.com/mudler/LocalAI/releases
[^6]: LocalAI README quickstart, macOS unsigned DMG note (issue #6268). https://github.com/mudler/LocalAI/issues/6268

## Tags

go, llm, inference-server, openai-compatible, self-hosted, local-inference, multimodal, llama-cpp, vllm, ai-api, edge-ai
