# ollama/ollama

The most-popular local-LLM runner — `ollama run modelname` and you have a chat or API endpoint against an open-weights model on your own machine.

## What it is

A Go-based CLI + daemon that pulls open-weights LLMs from the Ollama model registry, manages them on disk, and exposes a local HTTP API for inference. The defining UX is that `ollama run llama3` or `ollama run kimi-k2.5` "just works" — model conversion, quantization selection, GPU offload, and OS-specific runtime are abstracted. Built on llama.cpp internally for CPU/GPU inference; provides a streaming chat API compatible with the OpenAI client SDK shape.

## Key features

- One-command install + run: `ollama run <model>` pulls and starts the model.
- Model registry at ollama.com with hundreds of open-weights models (Llama family, Qwen, DeepSeek, Mistral, Gemma, Phi, MiniMax, GLM, gpt-oss, Kimi K2.5, and more).
- HTTP server with chat-completion + embeddings endpoints; OpenAI-compatible client surface.
- GPU acceleration via Metal (Apple), CUDA, and Vulkan; CPU fallback.
- macOS, Linux, and Windows installers; Docker image for containerized deployment.
- Modelfile system for customizing system prompts, sampling parameters, and adapter layers.

## Tech stack

- Go primary across the CLI, daemon, and HTTP server.
- llama.cpp as the inference engine (C/C++ via cgo).
- GGUF as the canonical model format.
- MIT-licensed.

## When to reach for it

- You want a local LLM running today, with one CLI command rather than picking through llama.cpp build flags.
- You're prototyping LLM features and want OpenAI-API-compatible local endpoints (so you can swap providers later).
- You're privacy-sensitive and need inference to stay on the host machine.

## When *not* to reach for it

- You need maximum throughput — vLLM, TGI, or SGLang are better for high-concurrency serving.
- You need fine-grained model conversion / quantization control — go to llama.cpp directly.
- Your hardware is too constrained — even 7B models need ~8GB RAM and benefit from a GPU.

## Maturity signal

173k stars, 16k forks, MIT, last push the morning this page was generated. 3-year-old project that became the default local-LLM runner essentially as soon as it shipped. The 3,300 open-issues count is high in absolute terms but tracks the breadth of supported models, OSes, and hardware — most are model-specific or driver-specific reports rather than core defects. Active investment from a small core team plus a large community.

## Alternatives

- `ggml-org/llama.cpp` — use when you want the underlying inference engine without the model-registry abstraction.
- LM Studio, Jan, Backyard AI — use when you want a graphical chat client rather than a CLI/daemon.
- vLLM, TGI, SGLang — use when you need high-concurrency serving for production.
- LocalAI — use when you want OpenAI-API compatibility across many backends.

## Notes

The "modelname picks the right quantization for your hardware" UX is what won Ollama its dominant position — users don't have to learn GGUF quant levels. Anyone building LLM tooling that defaults to local inference should expect their users to have Ollama installed already. License (MIT) and Go implementation make it embeddable into other tools.

## Tags

artificial-intelligence, large-language-model, llama, golang, local-inference, command-line-interface, model-registry, llama-cpp, gguf
