# p-e-w/heretic

A command-line tool that automatically removes safety-alignment refusals from transformer language models using directional ablation with automated parameter optimization.

## What it is

Heretic decensors ("abliterates") transformer-based language models without expensive post-training. It combines an implementation of directional ablation with a TPE-based parameter optimizer (Optuna) that searches for abliteration parameters by co-minimizing the number of refusals and the KL divergence from the original model, so the result stays close to the original model's behavior. It is aimed at anyone who can run a command-line program: no understanding of transformer internals is required. It also includes optional research features for studying the geometry of refusal directions.

## Key features

- Fully automatic operation: the optimizer finds abliteration parameters with no manual configuration, co-minimizing refusals and KL divergence from the original model.
- Broad model support: most dense models, many multimodal models, several MoE architectures, and some hybrid models such as Qwen3.5 (pure state-space and certain research architectures are not supported out of the box).
- A float-valued direction index that interpolates between the two nearest residual directions, opening a larger search space than the per-layer difference-of-means directions alone.
- Per-component ablation kernels chosen separately for attention out-projection and MLP down-projection, since MLP interventions tend to be more damaging.
- Optional `bnb_4bit` quantization via bitsandbytes to reduce VRAM, and an automatic batch-size benchmark at startup.
- After processing, options to save, upload to Hugging Face, chat-test, and run standard benchmarks; a built-in `--evaluate-model` mode reproduces the refusal/KL metrics.
- A `research` extra for interpretability work: PaCMAP residual-vector plots and animations (`--plot-residuals`) and a residual-geometry metrics table (`--print-residual-geometry`).

## Tech stack

- Python `>=3.10`, PyTorch `2.2+` (torch/torchvision versions deliberately unspecified; some models need later PyTorch, e.g. MXFP4 models use `torch.accelerator` from 2.6).
- `transformers[kernels]~=5.6`, `accelerate~=1.13`, `peft~=0.19`, `datasets~=4.7`, `huggingface-hub~=1.7`.
- Optimization via `optuna~=4.7`; quantization via `bitsandbytes~=0.49`; evaluation via `lm-eval[hf]~=0.4`; numerics via `numpy~=2.2`.
- CLI/UX via `rich`, `questionary`, `tqdm`, and `pydantic-settings`; configuration via TOML (`config.default.toml`).
- Research extra: `pacmap`, `matplotlib`, `scikit-learn`, `imageio`, `geom-median`.
- Built with the `uv` build backend and ships a `uv.lock` pinning every package version; AGPL-3.0-or-later; version 1.4.0, Beta.

## When to reach for it

- You want to remove refusals from an open-weight LLM without collecting data or running post-training.
- You want abliteration that damages the original model's capabilities as little as possible, measured by KL divergence.
- You want a reproducible, config- or CLI-driven workflow, including pinned dependencies via `uv`.
- You want to study refusal directions and residual geometry through the interpretability research features.

## When *not* to reach for it

- Your target is a pure state-space model or another unsupported research architecture.
- You lack a capable GPU: a 4B model takes about 20–30 minutes on an RTX 3090, and PaCMAP projection runs on CPU and can take an hour or more for larger models.
- The AGPL-3.0 license does not fit your intended closed-source or commercial redistribution.
- You want to train, fine-tune, or add safety alignment; Heretic's purpose is removing it, not model creation or moderation.

## Maturity signal

Created in September 2025, Heretic has gathered roughly 26.6k stars and pushes changes actively, with the last push landing the day before this snapshot. It is AGPL-3.0, not archived, and at version 1.4.0 with a Beta classifier and around 73 open issues — a young but established niche tool under active maintenance. A strong adoption signal is that the community has published well over 4,000 models made with Heretic on Hugging Face.

## Alternatives

- Use **mlabonne's AutoAbliteration** instead when you want a notebook-driven, more hands-on abliteration workflow.
- Use **FailSpy/abliterator** or **wassname's abliterator** instead when you want a lower-level library to script the ablation yourself.
- Use **ErisForge** instead when you want an alternative abliteration library with its own API.
- Use **deccp** instead when you specifically want that project's decensoring approach and dataset.

## Notes

- The README reports that on `google/gemma-3-12b-it`, the automatically generated Heretic model matches other abliterations' refusal suppression (3/100) at a substantially lower KL divergence (0.16 vs 0.45–1.04), which is the project's central quality claim.
- The maintainer states Heretic was written from scratch and reuses no code from the prior-art projects it lists.
- The residual direction index being a float rather than an integer — enabling interpolation between layer directions — is called out as one of the main innovations over existing abliteration systems.

## Tags

python, llm, transformer, abliteration, machine-learning, pytorch, cli, model-editing, interpretability
