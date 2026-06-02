# ggml-org/llama.cpp

The C/C++ LLM inference engine that made running LLMs on commodity hardware viable — the de facto runtime under Ollama, LM Studio, Jan, and most local-LLM tooling.

## What it is

A pure C/C++ implementation of LLM inference with quantization support, GPU offload, and an aggressive minimum-dependency stance. Authored by Georgi Gerganov; the broader `ggml` tensor library powers it. The single most important contribution: GGUF, the quantized model format that lets a 70B parameter model fit on consumer hardware. llama.cpp is the inference engine; the broader ecosystem (Ollama, LM Studio, KoboldCpp, and many more) wraps it.

## Key features

- CPU inference (AVX2/AVX-512/ARM NEON optimized) — no GPU required.
- GPU offload via CUDA, Metal (Apple), Vulkan, HIP (AMD), SYCL (Intel).
- Quantization formats: 2-bit, 3-bit, 4-bit, 5-bit, 6-bit, 8-bit, and mixed-precision (Q4_K_M is the popular default).
- GGUF format — single-file, metadata-rich, drop-in-replacement-friendly model files.
- HTTP server (`llama-server`) with OpenAI-compatible API.
- Speculative decoding, grammar-constrained generation, structured output via JSON schema.
- Bindings for Python, Node.js, Go, Rust, Swift, etc., layered on top of the C ABI.
- MIT-licensed.

## Tech stack

- C/C++ primary.
- ggml tensor library as the math substrate.
- Per-vendor GPU kernels written for each supported backend (CUDA, Metal, Vulkan, etc.).
- Minimal external dependencies — designed to compile cleanly across embedded and constrained environments.

## When to reach for it

- You're running LLM inference on CPU or consumer-GPU hardware and need maximum control over quantization.
- You're embedding LLM inference into an app where you can't bring in a Python runtime.
- You're a researcher experimenting with quantization formats — llama.cpp is where new formats land first.
- You're integrating with downstream tools (Ollama, LM Studio) and want to understand the substrate.

## When *not* to reach for it

- You need maximum throughput in production multi-user serving — vLLM, TGI, SGLang are closer-fit.
- You want a simple "run a model" UX without quantization decisions — Ollama wraps llama.cpp with sensible defaults.
- You want to fine-tune models — this is inference-only; training infrastructure is elsewhere.

## Maturity signal

114k stars, 19k forks, MIT, last push the day this page was generated. 3-year-old project that became the canonical local-LLM inference engine essentially as it shipped. Open-issues count of 1,715 tracks the breadth of hardware/quantization combinations. Project is healthy: rapid release cadence, large active contributor base, and underpins most local-LLM tooling.

## Alternatives

- `ollama/ollama` — use when you want a friendly wrapper around llama.cpp.
- `vllm-project/vllm`, `huggingface/text-generation-inference`, `sgl-project/sglang` — production high-throughput serving.
- `mlc-ai/mlc-llm` — use when you want WebGPU / browser inference.
- `mlx-explore/mlx` — use for Apple Silicon-specific inference.

## Notes

GGUF is the project's most important external artifact — model creators (Bartowski, TheBloke, MaziyarPanahi, mradermacher, etc.) publish GGUF quantizations of new model releases within hours of model launch, and llama.cpp is the runtime that consumes them. License (MIT) plus the no-Python stance make this the runtime to embed when shipping LLMs in iOS/Android apps or embedded systems.

## Tags

artificial-intelligence, large-language-model, llama, c-plus-plus, local-inference, ggml, gguf, quantization, llama-cpp, inference-engine
