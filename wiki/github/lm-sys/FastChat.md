# lm-sys/FastChat

> The training/serving/evaluation stack behind Vicuna, MT-Bench, and Chatbot Arena — historically important, now largely in maintenance mode.

[GitHub repo](https://github.com/lm-sys/FastChat) ·
[Demo (LMArena)](https://lmarena.ai/) ·
[License: Apache-2.0](https://github.com/lm-sys/FastChat/blob/main/LICENSE)

## Overview

FastChat is an open platform from LMSYS (the UC Berkeley / Stanford group) for training, serving, and evaluating LLM-based chatbots[^1]. It reached prominence in March 2023 as the release vehicle for **Vicuna**, a Llama fine-tune that was widely reported as reaching "90% of ChatGPT quality," and it became the reference stack for the open-weight chatbot wave that followed. The same repository ships **MT-Bench** (an LLM-as-a-judge multi-turn benchmark) and the original **Chatbot Arena** side-by-side battle UI, which collected over 1.5M human preference votes and powered the LLM Elo leaderboard[^1].

Its defining tension is that FastChat is three loosely-coupled products in one repo — a fine-tuning harness, a multi-model serving system, and an evaluation toolkit — and none of them is the current best-in-class at its job. The serving layer's default worker is a thin wrapper over `huggingface/transformers`, which is correct but slow; the throughput story now points elsewhere (vLLM, SGLang). The value that endures is FastChat as *glue*: an OpenAI-compatible facade over many model backends, a large registry of conversation/prompt templates, and the canonical MT-Bench implementation.

As of 2026 the project is effectively in maintenance mode. Stars (~39.5k) and forks (~4.8k) reflect its 2023–2024 importance, but the last substantive push was mid-2026 and the ~1,000 open issues signal that active LMSYS engineering has moved to the standalone lmarena.ai platform and to SGLang[^2]. Treat FastChat as a stable, well-understood tool for what it already does, not as a project tracking the frontier.

## Getting Started

```bash
pip3 install "fschat[model_worker,webui]"
```

Single-model chat in the terminal (downloads weights from Hugging Face on first run):

```bash
python3 -m fastchat.serve.cli --model-path lmsys/vicuna-7b-v1.5
# ~14GB VRAM for 7B; add --load-8bit to roughly halve it,
# or --device cpu / --device mps for no-GPU / Apple Silicon
```

The distributed serving stack is three processes — a controller, one or more model workers, and an OpenAI-compatible API server:

```bash
python3 -m fastchat.serve.controller
python3 -m fastchat.serve.model_worker --model-path lmsys/vicuna-7b-v1.5
python3 -m fastchat.serve.openai_api_server --host 0.0.0.0 --port 8000
```

Once running, it is a drop-in `http://localhost:8000/v1` endpoint for the `openai` SDK[^3].

## Architecture / How It Works

The serving system is deliberately decomposed into three roles that talk over HTTP:

1. **Controller** — a registry and router. Model workers register themselves with the models they host; the controller dispatches incoming requests to a worker that serves the requested model, with basic load balancing across replicas.
2. **Model worker** — loads and runs one or more models. The default worker uses `transformers` generation, which maximizes compatibility (LLaMA, Vicuna, ChatGLM, Falcon, T5, and dozens more) at the cost of throughput. Alternative workers exist for vLLM, and there are device backends for CUDA, CPU (AVX512/AMX), Apple MPS, Intel XPU, and Ascend NPU.
3. **Server frontends** — a Gradio web UI (`gradio_web_server`), the multi-tab Arena UI (`gradio_web_server_multi`), and the `openai_api_server` that presents the OpenAI REST contract.

A quietly load-bearing piece is `fastchat/conversation.py`: a large table of per-model prompt templates (system prompt, role markers, separators, stop strings). For a long stretch of 2023–2024 this file was the de-facto canonical source that many other projects vendored to format prompts correctly, because base tokenizers did not yet carry chat templates. That role has eroded now that models ship `chat_template` in `tokenizer_config.json`, and FastChat's hand-maintained templates can lag or diverge from a model's official formatting.

The evaluation side (`fastchat/llm_judge`) implements MT-Bench: a fixed set of multi-turn questions scored by a strong judge model (originally GPT-4), producing single-answer grades and pairwise win rates. The Chatbot Arena code turns pairwise human votes into an Elo/Bradley-Terry rating; the methodology is written up in the 2024 arena technical report[^4].

## Production Notes

**The default worker is not a high-throughput server.** The `transformers`-based `model_worker` does per-request generation with limited batching. For real serving load you either point FastChat at a vLLM worker or, more commonly today, skip FastChat's serving entirely and use vLLM or SGLang directly. FastChat's serving value is multi-model routing and the OpenAI facade, not tokens/sec.

**Conversation-template drift is a real footgun.** If you serve a newer model through FastChat and rely on its built-in template, verify the formatting against the model card. A mismatched system prompt or separator degrades output quality in ways that are easy to miss and hard to attribute.

**Three processes, three failure modes.** The controller/worker/frontend split means a worker can silently fail to register (wrong `--controller` URL or port), and the UI will simply show no models. The README's own guidance — reboot the Gradio server if models don't appear — is an accurate description of the operational experience. In multi-worker deployments you must assign distinct GPUs (`CUDA_VISIBLE_DEVICES`), ports, and `--worker` URLs by hand.

**Maintenance status matters for dependency pinning.** With ~1,000 open issues and a slowing commit cadence, do not expect prompt fixes for new-model support or transformers-version breakage. Pin `fschat`, `transformers`, and `torch` together; upgrading `transformers` underneath FastChat is the most common source of breakage because the worker calls into fast-moving internal generation APIs.

**Weights and licensing are separate from the code.** FastChat is Apache-2.0, but the models you run through it are not. Vicuna weights inherit Llama's license; you are responsible for the license of whatever `--model-path` you load. The ShareGPT data used to train Vicuna is not distributed with the repo.

## When to Use / When Not

**Use when:**
- You want a quick OpenAI-compatible local endpoint over many different HF models without writing serving code.
- You need MT-Bench specifically, or want to reproduce/extend Chatbot Arena methodology.
- You are studying or reproducing the Vicuna / open-chatbot lineage.
- You want one place that already knows the prompt template for an older model.

**Avoid when:**
- You need production inference throughput — reach for vLLM or SGLang.
- You are serving current frontier models and want templates guaranteed to match the model card; use the tokenizer's own `chat_template`.
- You need an actively maintained project tracking the newest models within days of release.
- You want a batteries-included local chat UI for casual use; a dedicated UI is friendlier.

## Alternatives

- vllm-project/vllm — use instead when serving throughput and latency matter; PagedAttention gives far higher tokens/sec, and it also exposes an OpenAI-compatible server.
- sgl-project/sglang — use instead for high-performance serving with structured/programmatic generation; it is LMSYS's own newer engine and now underpins their arena work.
- huggingface/text-generation-inference — use instead when you want a production-hardened, Hugging Face-supported serving container.
- oobabooga/text-generation-webui — use instead when you want a feature-rich local chat/UI for experimentation rather than a serving backend.
- EleutherAI/lm-evaluation-harness — use instead when you need broad standardized benchmarks rather than MT-Bench's LLM-judge multi-turn format.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2023-03-19 | Release vehicle for the LMSYS chatbot stack[^1]. |
| Vicuna | 2023-03-30 | Llama fine-tune, "90% of ChatGPT quality" framing[^5]. |
| Chatbot Arena | 2023-05 | Side-by-side LLM battles, Elo leaderboard[^1]. |
| MT-Bench | 2023-06 | Multi-turn LLM-as-a-judge benchmark[^6]. |
| LongChat | 2023-06 | 32K-context chatbots + long-context eval. |
| Vicuna v1.5 | 2023-08 | Rebased on Llama 2, 4K and 16K context. |
| LMSYS-Chat-1M | 2023-09 | 1M real-world conversation dataset[^7]. |
| Arena report | 2024-03 | Chatbot Arena technical report, arXiv:2403.04132[^4]. |

## References

[^1]: FastChat README — project overview, Chatbot Arena scale, feature list. https://github.com/lm-sys/FastChat
[^2]: Repository activity as of 2026-07 (last push 2026-05-01, ~1,000 open issues), fetched via GitHub API; LMSYS's active serving/arena development has moved to SGLang and lmarena.ai. https://github.com/lm-sys/FastChat
[^3]: FastChat OpenAI-compatible API docs. https://github.com/lm-sys/FastChat/blob/main/docs/openai_api.md
[^4]: Chiang et al., "Chatbot Arena: An Open Platform for Evaluating LLMs by Human Preference" — 2024. https://arxiv.org/abs/2403.04132
[^5]: LMSYS blog, "Vicuna: An Open-Source Chatbot Impressing GPT-4 with 90% ChatGPT Quality" — 2023-03-30. https://lmsys.org/blog/2023-03-30-vicuna/
[^6]: Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" — 2023. https://arxiv.org/abs/2306.05685
[^7]: Zheng et al., "LMSYS-Chat-1M: A Large-Scale Real-World LLM Conversation Dataset" — 2023. https://arxiv.org/abs/2309.11998

## Tags

python, llm, model-serving, llm-evaluation, chatbot, openai-compatible-api, vicuna, chatbot-arena, mt-bench, inference, maintenance-mode
