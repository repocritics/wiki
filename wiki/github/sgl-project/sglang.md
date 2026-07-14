# sgl-project/sglang

> A high-throughput LLM and multimodal serving engine built around KV-cache reuse (RadixAttention), plus a structured-generation frontend language.

[GitHub repo](https://github.com/sgl-project/sglang) ·
[Official website](https://sglang.io) ·
[License: Apache-2.0](https://github.com/sgl-project/sglang/blob/main/LICENSE)

## Overview

SGLang is two things bolted together under one name. The first is **SGLang Runtime (SRT)** — a GPU inference server for large language and multimodal models, competing directly with vLLM and TensorRT-LLM. The second is **SGLang the language** — a Python-embedded DSL for expressing multi-call LLM programs (branching, parallelism, constrained decoding) that the runtime can co-optimize[^1]. The project started at LMSYS (the group behind Chatbot Arena and Vicuna) in early 2024[^2], and by 2025 the runtime had become the more consequential half: it joined the PyTorch ecosystem[^3], took an a16z Open Source AI Grant, and is now used as a production and RL-rollout backend at large scale.

The defining idea is **RadixAttention**: KV-cache entries for request prefixes are held in a radix tree so that shared prefixes (system prompts, few-shot examples, agent scratchpads, tree-of-thought branches) are matched and reused across requests automatically, without the caller declaring anything[^1]. This is the same problem PagedAttention addresses at the block level, but SGLang pushes it up to cross-request prefix sharing, which is where its throughput wins on prompt-heavy and agentic workloads come from.

The tension: SGLang is fast-moving and hardware-coupled. It ships day-0 support for new model families (DeepSeek, Qwen, GLM, Llama, gpt-oss) and new accelerators (Hopper, Blackwell, MI300/MI355, TPU, Ascend) often before the kernels stabilize. The cost is a large, churning dependency surface (`torch`, `flashinfer`, `sgl-kernel`) and a high open-issue count that reflects how many bleeding-edge configurations it tries to support at once.

## Getting Started

```bash
pip install "sglang[all]"
# CUDA kernels (flashinfer, sgl-kernel) are pulled in; match your torch/CUDA
```

Launch an OpenAI-compatible server:

```bash
python -m sglang.launch_server \
  --model-path meta-llama/Llama-3.1-8B-Instruct \
  --port 30000
```

```python
# The server speaks the OpenAI API — no SGLang-specific client needed
from openai import OpenAI
client = OpenAI(base_url="http://localhost:30000/v1", api_key="none")
resp = client.chat.completions.create(
    model="default",
    messages=[{"role": "user", "content": "Explain RadixAttention in one sentence."}],
)
print(resp.choices[0].message.content)
```

The frontend DSL is a separate entry point — structured programs with parallel forks and constrained output:

```python
import sglang as sgl

@sgl.function
def multi_turn(s, question):
    s += sgl.user(question)
    s += sgl.assistant(sgl.gen("answer", max_tokens=256))
    # forks share the prefix KV cache via RadixAttention
```

## Architecture / How It Works

The runtime is a Python orchestration layer over CUDA/HIP kernels:

- **RadixAttention + prefix cache.** Cached KV blocks are indexed by a radix tree keyed on token prefixes. Incoming requests are matched against the longest existing prefix and only the new suffix is computed. Eviction is LRU over tree leaves. Cache-aware routing extends this across replicas so requests with shared prefixes land on the same worker[^4].
- **Zero-overhead / overlap scheduler.** The CPU-side batch scheduler runs one step ahead of the GPU so that scheduling, tokenization, and cache bookkeeping overlap with kernel execution rather than stalling it — the "zero-overhead batch scheduler" from v0.4[^4].
- **Continuous batching + chunked prefill.** Requests join and leave the running batch per step; long prompts are prefilled in chunks so they don't block decode latency for other requests.
- **Attention backends.** Attention is delegated to pluggable kernels — FlashInfer is the common default on NVIDIA, with FlashAttention and Triton paths, plus vendor backends (AITER on AMD). The backend choice is a launch flag and materially changes performance and correctness on edge models.
- **Parallelism.** Tensor, pipeline, data, and expert parallelism are supported, including prefill-decode (PD) disaggregation and large-scale expert parallelism for MoE models across many GPUs/nodes.
- **Constrained decoding.** Structured outputs (JSON/regex/grammar) use a compressed finite-state-machine approach that can emit multiple deterministic tokens per step instead of one-at-a-time masking[^5].

The frontend language compiles a program into a call/dependency graph the runtime executes, letting it batch and cache-share across the program's LLM calls. In practice most users touch only the OpenAI-compatible server and never write DSL programs.

## Production Notes

**Installation is the first footgun.** SGLang pins tight version ranges for `torch`, `flashinfer`, and its own `sgl-kernel`. Mismatched CUDA/torch/driver combinations produce import errors or silent kernel fallbacks. Prefer the project's Docker images over a bare `pip install` for reproducible deploys, and pin the SGLang version explicitly — minor releases change kernel requirements.

**Memory tuning is manual.** `--mem-fraction-static` governs how much VRAM is reserved for the KV cache pool versus weights and activations. Set it too high and you OOM under concurrency; too low and you cap throughput. Long-context and large-batch workloads usually need this tuned per model and per GPU rather than left at default.

**RadixAttention helps unevenly.** Workloads with heavy shared prefixes (agents, few-shot, shared system prompts, beam/tree search) see large gains. Workloads of unique, unrelated prompts get little prefix reuse — the radix tree becomes overhead you pay without the payoff. Measure on your own traffic before assuming the headline numbers apply.

**Bleeding-edge model support carries risk.** Day-0 support for a new architecture may ship with a specific attention backend, quantization format, or parallelism mode as the only working path; other combinations can be broken or slower until follow-up PRs land. Check the model's known-good launch flags rather than assuming every feature composes.

**Churn and issue volume.** The project releases frequently and carries a large open-issue backlog. Many issues are environment/hardware-specific rather than core bugs, but the corollary is that upgrading across minor versions can change defaults (schedulers, backends, cache behavior). Treat version bumps as changes to validate, not drop-in.

**Non-NVIDIA paths are real but less trodden.** AMD (ROCm), TPU (SGLang-Jax), Intel, and Ascend backends exist and are actively developed, but coverage and performance parity lag the CUDA path — verify your exact model + hardware combination is on the tested list.

## When to Use / When Not

**Use when:**
- You serve prompt-heavy or agentic traffic with large shared prefixes and want automatic KV reuse.
- You need day-0 support for a newly released open model on current NVIDIA/AMD hardware.
- You want an OpenAI-compatible endpoint with high concurrent throughput and structured-output decoding.
- You're building an RL post-training loop and need a fast, scriptable rollout/generation backend.

**Avoid when:**
- You want a stable, rarely-changing dependency — SGLang iterates fast and pins hard.
- Your workload is low-QPS or single-user local chat; a lighter runtime (Ollama, llama.cpp) is simpler.
- You need a mature, vendor-supported path on non-NVIDIA hardware today.
- You need one framework across cloud and constrained edge devices — SGLang targets datacenter GPUs.

## Alternatives

- vllm-project/vllm — the closest peer; PagedAttention, broad model support, larger community. Use it when you want the most widely deployed OSS engine and a slightly calmer release cadence.
- NVIDIA/TensorRT-LLM — fastest on NVIDIA when you can afford ahead-of-time engine compilation and NVIDIA-only lock-in.
- huggingface/text-generation-inference — use when you want tight Hugging Face Hub integration and a supported, less experimental stack.
- ggml-org/llama.cpp — use for CPU/edge/single-machine and GGUF quantized local inference, not datacenter throughput.
- ollama/ollama — use for local developer UX and easy model pulls; not a high-concurrency serving engine.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2024-01 | First release; RadixAttention, structured-generation frontend[^1]. |
| v0.2 | 2024-07 | Runtime perf pass; Llama 3 serving benchmarks vs TRT-LLM/vLLM[^6]. |
| v0.3 | 2024-09 | Faster DeepSeek MLA, torch.compile path, multi-image/video support[^7]. |
| v0.4 | 2024-12 | Zero-overhead batch scheduler, cache-aware load balancer, faster structured output[^4]. |
| — | 2025-01 | Day-0 DeepSeek V3/R1 support on NVIDIA and AMD. |
| — | 2025-03 | Joined the PyTorch ecosystem[^3]; a16z Open Source AI Grant. |

## References

[^1]: Zheng et al., "SGLang: Efficient Execution of Structured Language Model Programs" — arXiv:2312.07104. https://arxiv.org/abs/2312.07104
[^2]: SGLang blog, "Fast and Expressive LLM Inference with RadixAttention and SGLang" — 2024-01-17. https://lmsys.org/blog/2024-01-17-sglang/
[^3]: PyTorch blog, "SGLang Joins PyTorch Ecosystem: Efficient LLM Serving Engine" — 2025-03. https://pytorch.org/blog/sglang-joins-pytorch/
[^4]: SGLang blog, "v0.4 Release: Zero-Overhead Batch Scheduler, Cache-Aware Load Balancer, Faster Structured Outputs" — 2024-12-04. https://lmsys.org/blog/2024-12-04-sglang-v0-4/
[^5]: SGLang blog, "Fast JSON Decoding for Local LLMs with Compressed Finite State Machine" — 2024-02-05. https://lmsys.org/blog/2024-02-05-compressed-fsm/
[^6]: SGLang blog, "Achieving Faster Open-Source Llama3 Serving with SGLang Runtime" — 2024-07-25. https://lmsys.org/blog/2024-07-25-sglang-llama3/
[^7]: SGLang blog, "v0.3 Release" — 2024-09-04. https://lmsys.org/blog/2024-09-04-sglang-v0-3/

## Tags

python, cuda, llm-inference, model-serving, radixattention, kv-cache, structured-generation, moe, multimodal, gpu, lmsys, high-throughput
