# GeeeekExplorer/nano-vllm

A lightweight vLLM implementation built from scratch in roughly 1,200 lines of Python.

## What it is

Nano-vLLM is a from-scratch reimplementation of the vLLM offline inference engine, aimed at people who want to read and understand how a modern LLM inference stack works without wading through a large production codebase. It keeps the core in about 1,200 lines of Python while still applying the optimizations that make batched inference fast. The API deliberately mirrors vLLM's, so someone familiar with vLLM can pick it up with minimal changes.

## Key features

- Offline batch inference with speeds the README reports as comparable to vLLM.
- Compact, readable implementation of around 1,200 lines of Python.
- Optimization suite covering prefix caching, tensor parallelism, Torch compilation, and CUDA graphs.
- API that mirrors vLLM's, with `LLM` and `SamplingParams` entry points and a `generate` method that differs only slightly.
- Installable directly from the Git repository via pip.

## Tech stack

- Python, constrained to `>=3.10,<3.13`.
- PyTorch (`torch>=2.4.0`) as the tensor and compilation backend.
- Triton (`triton>=3.0.0`) for GPU kernels.
- Hugging Face Transformers (`transformers>=4.51.0`) for model and tokenizer support.
- `flash-attn` for attention, and `xxhash` as a hashing dependency.
- Packaged with setuptools (`setuptools>=61`); current project version 0.2.0.

## When to reach for it

- You want to study or teach how a vLLM-style inference engine works internally.
- You need offline, batched text generation on a GPU and want a small codebase you can modify.
- You are running Qwen3-family models (the shipped example and benchmark use Qwen3-0.6B).
- You already know vLLM's API and want a lighter drop-in for local experimentation.

## When *not* to reach for it

- You need an online serving stack with a request server, streaming endpoints, or scheduling for production traffic; the README describes offline inference only.
- You depend on broad model-architecture coverage. The repository ships a single model file (`nanovllm/models/qwen3.py`), so support beyond Qwen3 is not evident from the inputs.
- You want the full breadth of features and hardware support that the upstream vLLM project maintains.
- You cannot meet the GPU-oriented dependencies such as `flash-attn`, Triton, and CUDA graphs.

## Maturity signal

Created in mid-2025 and pushed to within roughly three months of this writing, Nano-vLLM is recently active rather than dormant. It has drawn an unusually large audience for a young project, with over 14,000 stars and more than 2,000 forks, alongside around 78 open issues that reflect that attention. The MIT license and 0.2.0 version number indicate a permissively licensed, still-evolving project that is well past a first sketch but has not declared API stability.

## Alternatives

- vLLM — reach for it instead when you need a production-grade serving engine with broad model and hardware support rather than a minimal, readable core.
- SGLang — use it when you want a high-throughput serving framework with structured generation features beyond basic batch inference.
- Hugging Face Transformers `generate` — use it when you prioritize the widest model coverage and simplicity over batched-inference throughput.
- llama.cpp — use it when you need CPU or mixed-hardware inference and quantized model formats rather than a PyTorch and CUDA stack.

## Notes

In the README's own benchmark on an RTX 4070 Laptop (8GB) with Qwen3-0.6B over 256 sequences, Nano-vLLM finishes slightly ahead of vLLM (1434.13 vs 1361.84 tokens/s) for identical output token counts, which is notable given that it fits its engine in about 1,200 lines.

## Tags

python, machine-learning, llm, inference, pytorch, deep-learning, transformer, tensor-parallelism, prefix-caching, library
