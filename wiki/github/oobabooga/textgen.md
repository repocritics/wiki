# oobabooga/textgen

> A desktop app and Gradio web UI for running local LLMs — the long-running "oobabooga" front end, now spanning text, vision, tool-calling, and an OpenAI/Anthropic-compatible API.

[GitHub repo](https://github.com/oobabooga/textgen) ·
[License: AGPL-3.0](https://github.com/oobabooga/textgen/blob/main/LICENSE)

## Overview

TextGen (formerly `text-generation-webui`, still redirected from that name) is one of the oldest surviving front ends for running large language models on your own hardware. It was created in December 2022 and rose to prominence in early 2023 as a primary UI for the leaked LLaMA weights, explicitly modeled on AUTOMATIC1111's Stable Diffusion web UI — the goal being "the same thing, but for text"[^1]. For most of its life it was a browser-based Gradio app started from a Python script; recent releases add portable, dependency-bundled desktop builds and a rebrand to "TextGen".

The defining characteristic is **breadth of backends behind one interface**. Rather than committing to a single inference engine, it wraps several loaders — llama.cpp (GGUF), ik_llama.cpp, Hugging Face Transformers, ExLlamaV3, and TensorRT-LLM — and lets you switch models and backends without restarting[^2]. This makes it a convenient testbed for quantization formats, samplers, and prompt templates, and it is where many local-LLM users first learn what parameters like `top_k`, `min_p`, `mirostat`, or `DRY` actually do.

That same breadth is the project's central tension. The surface area is enormous (the CLI alone exposes well over a hundred flags), the loaders have divergent capabilities and quirks, and the code has been through several architectural rewrites as the local-inference landscape churned. It is a hobbyist and researcher tool optimized for tinkering and privacy, not a hardened multi-tenant serving stack. The license is AGPL-3.0, which has practical consequences for anyone considering embedding it in a hosted product.

## Getting Started

The recommended path is a portable build (download, unzip, run) from the releases page. For a source install with a virtualenv:

```bash
git clone https://github.com/oobabooga/textgen
cd textgen
python -m venv venv && source venv/bin/activate     # Windows: venv\Scripts\activate
pip install -r requirements/portable/requirements.txt --upgrade
python server.py --portable --api --auto-launch
```

Place a GGUF file under `user_data/models/` and it is auto-detected. With `--api`, an OpenAI-compatible server runs alongside the UI:

```bash
curl http://127.0.0.1:5000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"local","messages":[{"role":"user","content":"Say hello."}]}'
```

The full installation (`start_windows.bat` / `start_linux.sh` / `start_macos.sh`) sets up a Conda environment via Miniforge and pulls PyTorch; it is required for Transformers/ExLlamaV3 loaders, LoRA training, image generation, and extensions, and needs roughly 10 GB of disk.

## Architecture / How It Works

At the core is `server.py`, which boots a Gradio app plus an optional API server. The UI is organized into tabs — chat, notebook, parameters, model, training, and (recently) image generation — with state held in Python globals rather than a formal application model. Chat runs in `instruct`, `chat`, or `chat-instruct` modes, and prompts are formatted through Jinja2 templates so a model's own chat template can be applied[^2].

The loader layer is the real substance. Each backend is a thin adapter over an external engine:

- **llama.cpp / ik_llama.cpp** — GGUF models, the default and lowest-friction path. GPU offload is controlled per-layer (`--gpu-layers`), with options for KV-cache quantization, MoE expert offload to CPU, tensor split across GPUs, and speculative decoding (draft models and n-gram drafting).
- **Transformers** — full-precision or bitsandbytes 4/8-bit models, the widest model compatibility and the slowest common path. Gated behind `--trust-remote-code` for custom modeling code.
- **ExLlamaV3** — quantized (EXL3) inference optimized for consumer NVIDIA GPUs.
- **TensorRT-LLM** — NVIDIA's compiled-engine path for maximum throughput on supported hardware.

Sampling is unusually rich: beyond the standard temperature/top-p/top-k, it exposes dynamic temperature, mirostat, XTC, DRY repetition penalty, top-n-sigma, and sampler-priority reordering — many surfaced before they appeared in other UIs. The **OpenAI/Anthropic-compatible API** exposes Chat, Completions, and Messages endpoints with tool-calling, where each tool is a single `.py` file and MCP servers can also be attached[^2]. **Extensions** are Python modules loaded at startup (TTS, voice input, translation, and community add-ons); because they run in-process with full trust, they are effectively plugins, not sandboxed.

Because every loader is an external dependency compiled for specific CUDA/ROCm/Vulkan/CPU targets, most of the project's real complexity lives in its precompiled requirements wheels and installer scripts rather than in application logic.

## Production Notes

**This is not a production serving framework, and does not pretend to be.** For concurrent, high-throughput, multi-tenant serving, purpose-built engines (vLLM, TGI, TensorRT-LLM directly) are the correct tools; TextGen is single-user-first and its multi-user mode is described as suitable only for small trusted teams.

- **AGPL-3.0.** Network use triggers the AGPL's source-disclosure obligation. If you expose it as a hosted service, the license reaches your modifications. Weigh this before building anything commercial on top of it — many teams pick a permissively licensed backend instead.
- **Dependency and install fragility.** The most common failure mode is environment breakage: mismatched CUDA/PyTorch/loader wheel versions, ROCm builds, and extension dependencies clobbering core requirements. The portable builds exist specifically to sidestep this; the full Conda install is where most support issues originate. Updating requirements can silently pull incompatible wheels.
- **`--trust-remote-code` and extensions execute arbitrary code.** Loading some Transformers models, and any extension, runs untrusted Python with your privileges. Treat model repos and extensions as code, not data.
- **Exposure flags are footguns.** `--listen` binds to all interfaces, `--share` opens a public Gradio tunnel, and the API defaults to no auth unless `--api-key`/`--admin-key` are set. Do not expose an instance to a network without authentication and a reverse proxy.
- **VRAM is the real constraint.** Loading behavior, context size, and KV-cache type interact non-obviously; OOM on model load or at long context is routine. The community GGUF VRAM calculators exist because this is the recurring pain point.
- **Churn.** The project has undergone multiple internal rewrites and has dropped loaders as the ecosystem shifted (older GPTQ/GGML/ExLlamaV2 paths gave way to GGUF and ExLlamaV3). Pinning a known-good commit is common practice for anyone scripting against it.

## When to Use / When Not

**Use when:**
- You want to run local models fully offline with no telemetry and a real GUI.
- You need to compare backends, quantization formats, and samplers interactively on one machine.
- You want an OpenAI/Anthropic-compatible endpoint in front of a local GGUF model for development.
- You are experimenting with LoRA fine-tuning, vision models, or tool-calling on your own hardware.

**Avoid when:**
- You need concurrent, high-throughput, or multi-tenant serving (use a dedicated inference server).
- AGPL-3.0 is incompatible with your distribution or hosting plans.
- You want a minimal, headless, single-purpose API server — the Gradio UI and its dependencies are overhead you don't need.
- You want turnkey model management and a stable, narrow surface — a simpler tool will have fewer moving parts.

## Alternatives

- ggml-org/llama.cpp — the underlying GGUF inference engine and its own `llama-server`; use it directly when you want the backend without a Gradio UI.
- ollama/ollama — opinionated model pull/run plus a simple API; use it when you value simplicity over configurability and sampler depth.
- LostRuins/koboldcpp — single-binary llama.cpp front end oriented toward storytelling/roleplay; use it when you want zero-install and a narrower feature set.
- open-webui/open-webui — a polished chat front end that talks to a separate backend (Ollama or any OpenAI-compatible API); use it when you want the UI decoupled from inference.
- vllm-project/vllm — throughput-oriented serving engine for GPUs; use it when you are serving many concurrent requests in production, not tinkering locally.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Initial release | 2022-12 | Gradio web UI for local text generation, modeled on the Stable Diffusion web UI[^1]. |
| LLaMA-era adoption | 2023 | Became a primary UI for local LLaMA/derivative models; one-click installers, multiple loaders (Transformers, GPTQ, ExLlama, llama.cpp). |
| GGUF consolidation | 2023–2024 | llama.cpp/GGUF became the default low-friction path as older quant loaders were dropped. |
| OpenAI-compatible API | 2024 | Built-in OpenAI-style Chat/Completions endpoints for local drop-in use. |
| Desktop rebrand ("TextGen") | 2025–2026 | Portable dependency-bundled builds, Anthropic-compatible Messages API, tool-calling/MCP, vision, and an image-generation tab. |

## References

[^1]: `oobabooga/textgen` README and project history — origin as a text-generation counterpart to AUTOMATIC1111's Stable Diffusion web UI. https://github.com/oobabooga/textgen
[^2]: `oobabooga/textgen` README — backends, modes, API, tool-calling, and extensions. https://github.com/oobabooga/textgen

## Tags

python, local-llm, llm, gradio, llama-cpp, gguf, inference-ui, openai-compatible-api, self-hosted, agpl, desktop-app
