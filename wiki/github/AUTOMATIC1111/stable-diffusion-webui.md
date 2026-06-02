# AUTOMATIC1111/stable-diffusion-webui

The mainline Gradio-based web UI for running Stable Diffusion locally — the de facto reference frontend during the 2022-2024 SD wave.

## What it is

A Gradio web application that wraps Stable Diffusion (and other diffusion models) into a browsable, batteries-included UI: txt2img, img2img, inpainting, outpainting, upscaling, ControlNet/extensions, LoRA training, model and embedding management. Defined the UX vocabulary that downstream forks (Forge, ReForge, vladmandic, SD.Next) all inherit. AGPL-3.0 licensed.

## Key features

- txt2img / img2img / inpainting / outpainting workflows under a single Gradio UI.
- Extension architecture — ControlNet, RegionalPrompter, LoRA, embeddings, hypernetworks plug in via the `extensions/` directory.
- Upscaling pipeline with multiple algorithms (ESRGAN, SwinIR, LDSR).
- Prompt-history, image-info round-trip, X/Y plot for parameter sweeps.
- CLI args + `webui-user.bat` / `webui-user.sh` for environment customization.
- Cross-platform support (Windows, Linux, macOS) with GPU-vendor coverage (NVIDIA, AMD, Intel, Apple Silicon).

## Tech stack

- Python primary; PyTorch for inference.
- Gradio for the UI layer; bundles its own Gradio fork in some releases.
- AGPL-3.0 licensed — network-service redistribution requires source release.

## When to reach for it

- You want a feature-complete local SD interface and don't mind installing dependencies.
- You're using extensions that target this specific UI's plugin API (ControlNet+SD15 ecosystem).
- You want a baseline to compare other Diffusion UIs against.

## When *not* to reach for it

- You want SDXL/SD3/Flux as your primary model — ComfyUI's node-graph UI has become the preferred frontend for newer model families.
- You want a code-first / programmatic interface — Diffusers library is closer-fit.
- You're allergic to AGPL implications for SaaS hosting.

## Maturity signal

163k stars, 30k forks, AGPL-3.0, last push March 2026 — pace has slowed substantially from the 2023 peak. The 2,490 open-issues count tracks extension breakage and platform-specific quirks. Project has effectively entered "long-tail stewardship" mode as the diffusion-UI center of gravity moved to ComfyUI and Forge for newer models.

## Alternatives

- `Comfy-Org/ComfyUI` — node-graph UI; preferred for SDXL/SD3/Flux and complex pipelines.
- `lllyasviel/Forge` / `vladmandic/sdnext` — actively-maintained forks with performance optimizations.
- Hugging Face `diffusers` library — use when you want a Python API rather than a UI.
- InvokeAI — use when you want a polished UI with built-in canvas/inpainting workflow.

## Notes

The "AUTO1111" name remains the shorthand even as the project's release cadence slowed. Anyone setting up a new diffusion workstation today should weigh AUTO1111 against ComfyUI based on which models they target — AUTO1111 is solid for SD1.5/SD2 + classic extensions, ComfyUI for modern model families.

## Tags

artificial-intelligence, machine-learning, stable-diffusion, image-generation, python, pytorch, gradio, agpl, deep-learning, user-interface
