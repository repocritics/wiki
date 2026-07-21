# karpathy/autoresearch

A single-GPU harness that lets an AI coding agent autonomously edit and iterate on a small LLM training script overnight.

## What it is

autoresearch gives an AI agent a small but real LLM training setup and lets it experiment on its own: it modifies the code, trains for five minutes, checks whether the result improved, keeps or discards the change, and repeats. The training code is a simplified single-GPU implementation of nanochat. The intended workflow inverts the usual one — instead of editing the Python yourself, you program the `program.md` Markdown file that provides context to the agent and defines your autonomous research setup, while the agent edits the training code. It targets people who want to tinker with autonomous ML research experiments on a single GPU.

## Key features

- Training runs on a fixed five-minute wall-clock budget (excluding startup and compilation), which the README estimates at roughly 12 experiments per hour and around 100 overnight.
- The metric is `val_bpb` (validation bits per byte); lower is better, and it is vocab-size-independent so architectural changes are compared fairly.
- Only three files matter: `prepare.py` (constants, one-time data prep, runtime utilities — not modified), `train.py` (the single file the agent edits, containing the GPT model, a Muon + AdamW optimizer, and the training loop), and `program.md` (baseline agent instructions, edited by the human).
- You run it by pointing a coding agent (Claude, Codex, or similar) at `program.md` with all permissions disabled and prompting it to start an experiment.
- Self-contained by design: no dependencies beyond PyTorch and a few small packages, no distributed training, no complex configs.

## Tech stack

- Python, requires `>=3.10` (README states Python 3.10+).
- PyTorch pinned to `torch==2.9.1`, installed from the CUDA 12.8 wheel index (`pytorch-cu128`).
- `uv` as the project/dependency manager (`uv sync`, `uv run`).
- Dependencies: `kernels>=0.11.7`, `matplotlib>=3.10.8`, `numpy>=2.2.6`, `pandas>=2.3.3`, `pyarrow>=21.0.0`, `requests>=2.32.0`, `rustbpe>=0.1.0`, `tiktoken>=0.11.0`.
- Requires a single NVIDIA GPU (tested on H100).

## When to reach for it

- You want to run autonomous, overnight LLM-training experiments driven by a coding agent on one GPU.
- You want a minimal, reviewable training loop where the agent only touches a single file, keeping diffs small.
- You are learning or tinkering with LLM pretraining and want directly comparable experiments under a fixed time budget.
- You want to find the most performant model your specific platform can reach within the fixed budget.

## When *not* to reach for it

- You do not have a single NVIDIA GPU; the code currently requires one, and CPU/MPS support is explicitly not included.
- You need to run on macOS, Windows, or AMD hardware — the README points to community forks instead.
- You need distributed or multi-GPU training; there is intentionally none.
- You want results comparable to other people's runs — the fixed time budget makes results non-comparable across different compute platforms.
- You want a full-featured or production training system rather than a deliberately bare-bones baseline.

## Maturity signal

The repository was created on 2026-03-06 and last pushed on 2026-03-26, and it is not archived. In that short window it accumulated roughly 91,700 stars and 13,100 forks, reflecting very high interest driven largely by the author's profile rather than a long maturation. It is best read as a deliberately minimal reference project and starting point — small by design, recently active, and clearly meant to be forked and iterated on — not as a maturing or feature-complete product.

## Alternatives

- karpathy/nanochat — the full parent repository with wider platform support (a Flash Attention 3 fallback, generic device support, autodetection). Use it when you need broader platform coverage or a less stripped-down implementation.
- miolini/autoresearch-macos or trevin-creator/autoresearch-mlx — community forks listed for macOS. Use these when running on a Mac instead of an NVIDIA GPU.
- jsegov/autoresearch-win-rtx — a fork listed for Windows/RTX. Use it when running on Windows hardware.
- andyluo7/autoresearch — a fork listed for AMD. Use it when running on AMD GPUs.

## Notes

- The workflow inverts normal research practice: you edit the Markdown (`program.md`), which the README calls a "super lightweight skill", while the agent edits the Python.
- The README's opening is a satirical science-fiction framing (autonomous agent swarms in a "10,205th generation" self-modifying codebase), not a technical description.
- The `pyproject.toml` describes the project as an "Autonomous pretraining research swarm", a different phrasing than the GitHub description.
- The README states the license is MIT, but the repository's SPDX license metadata is null, so the machine-readable license field does not confirm it.
- The README includes detailed guidance for adapting the defaults to much smaller machines (lower `DEPTH`, `vocab_size`, `MAX_SEQ_LEN`, `TOTAL_BATCH_SIZE`, and a narrower TinyStories dataset), aimed at fork authors.

## Tags

python, machine-learning, llm, deep-learning, pytorch, ai-agents, autonomous-agents, gpu, pretraining
