# AlexsJones/llmfit

A terminal tool that detects your hardware and tells you which local LLM models will actually run well on it.

## What it is

llmfit is a Rust command-line and TUI application that right-sizes LLM models to your system's RAM, CPU, and GPU. It detects your hardware, then scores every model in its catalog across memory fit, estimated speed, quality, and context, so you can pick a model before downloading gigabytes that won't run. It targets developers and local-AI users who run models through providers like Ollama, llama.cpp, MLX, Docker Model Runner, and LM Studio and want a grounded answer to "what fits my machine."

## Key features

- Interactive TUI (default) plus a classic CLI mode with subcommands: `fit`, `recommend`, `info`, `bench`, and `doctor`.
- Hardware detection (RAM, CPU, GPU/VRAM, backend) with a `doctor` report intended for bug reports.
- Model scoring across four dimensions — memory fit, estimated speed, quality, and context — with support for multi-GPU setups, MoE architectures, and dynamic quantization selection.
- Speed estimates from a memory-bandwidth model grounded in runtime sampling and community measurements; `info` shows the inputs behind each estimate and how to verify it.
- On-machine benchmarking (`bench`) that measures real tok/s and TTFT against a running provider, with the option to contribute results back so identical hardware gets measured numbers.
- JSON output (`recommend --json`) for consumption by scripts and agents, plus a Docker image that prints `recommend` JSON by default.

## Tech stack

- Rust, organized as a Cargo workspace (`resolver = "3"`) with members `llmfit-core`, `llmfit-tui`, and `llmfit-desktop`; default members are core and TUI.
- Workspace version 1.1.6, MIT-licensed.
- Additional surfaces in the repo: a Python package (`llmfit-python`), a Tauri-based desktop app (`llmfit-desktop`), and a React/Vite web UI (`llmfit-web`).
- Distributed via Scoop, Homebrew, MacPorts, a `curl | sh` installer, `uv`/`pip`, Docker/Podman (`ghcr.io/alexsjones/llmfit`), and Cargo from source.

## When to reach for it

- Deciding which local model to download for a given machine without trial and error.
- Comparing quantizations and checking whether a model fits within available RAM/VRAM, including multi-GPU and MoE cases.
- Producing machine-consumable recommendations for scripts or agents via `recommend --json`.
- Benchmarking real throughput on your own hardware and sharing measurements back to the catalog.

## When *not* to reach for it

- You want to actually pull, serve, and chat with models — llmfit sizes and scores them; running them is the job of the providers it integrates with.
- You need coverage of a model that is not in the catalog and you are not prepared to add it (custom-model workflows exist but require setup).
- You are looking for cloud/hosted API cost or latency comparisons rather than local hardware fit.

## Maturity signal

Despite being created in early 2026, llmfit has drawn a very large following and pushes commits continuously (last push the same week as this writing), which points to an actively and heavily maintained project rather than an experiment. It has crossed the 1.x line (workspace version 1.1.6), ships signed Windows binaries through a release pipeline, and carries a permissive MIT license, so adoption and forking are low-risk. The combination of rapid recent activity and a young codebase means interfaces may still move, but the maintenance cadence is strong.

## Alternatives

- llm-checker — use instead when you want a Node.js CLI that actually pulls and runs models through Ollama to measure real-world performance, and you don't need MoE-aware memory estimates (the README notes it treats all models as dense).
- Ollama — use instead when your goal is simply to pull and run models locally and you don't need fit scoring or hardware analysis beforehand.
- LM Studio — use instead when you prefer a graphical app for discovering and running local models over a terminal-first workflow.

## Notes

- The project is deliberate about verifiability: every speed estimate ships its inputs, and `llmfit info` shows exactly what a number assumes plus the commands to confirm it on your machine.
- Community benchmarks are contributed as pull requests straight from the TUI (no `gh` CLI or third-party account), and merged submissions ship measured numbers in the next release.
- It states an explicit privacy stance: it only contacts external services when you use the corresponding feature (model downloads, provider queries, or the leaderboard).

## Tags

rust, cli, tui, llm, local-llm, machine-learning, developer-tools, gguf, quantization
