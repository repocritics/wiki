# Comfy-Org/ComfyUI

A node-graph UI for diffusion models — the canonical frontend for advanced Stable Diffusion / SDXL / SD3 / Flux pipelines.

## What it is

A Python-based application that exposes diffusion-model inference as a visual node graph: each node is a transformation (load checkpoint, encode prompt, sample, decode, upscale, save), and edges connect tensor outputs to inputs. The graph metaphor matches how diffusion pipelines actually work — multi-stage conditioning, ControlNet integration, LoRA composition, regional prompting, and complex sampling schedules all express naturally as nodes. Has surpassed AUTOMATIC1111 as the preferred UI for newer model families (SDXL, SD3, Flux, video diffusion).

## Key features

- Node-graph editor — diffusion pipelines compose visually with explicit data flow.
- First-class support for advanced workflows: ControlNet, LoRA stacking, regional prompting, conditioning math, multi-stage sampling.
- Custom-node ecosystem — community packs that add specialized capabilities (upscalers, video pipelines, IP-Adapter, FaceID, etc.).
- Workflow JSON files can be shared, versioned, and embedded in generated images for full reproducibility.
- API mode — same graphs can be invoked headlessly via HTTP for programmatic pipelines.
- GPL-3.0 licensed.

## Tech stack

- Python primary.
- PyTorch as the underlying inference framework.
- JavaScript on the renderer side for the node-graph editor.
- Custom node loading via Python entry points.

## When to reach for it

- You're working with SDXL, SD3, Flux, or video diffusion models that benefit from multi-stage pipelines.
- You need reproducible, shareable workflows where the graph is the source of truth.
- You're integrating diffusion into a larger pipeline and want headless API access alongside a GUI.

## When *not* to reach for it

- You want a simple one-prompt one-image UX — AUTOMATIC1111 or InvokeAI have lower friction for basic workflows.
- You're allergic to GPL-3.0 — the copyleft pulls derivative SaaS services into the same license.
- You don't want to manage custom-node compatibility — the community pack ecosystem has the usual long-tail breakage.

## Maturity signal

115k stars, 13k forks, GPL-3.0, last push the day this page was generated. 3-year-old project under the Comfy-Org organization (formalized in 2024 after the original maintainer attracted commercial backing). 4,000 open-issues count tracks the broad custom-node ecosystem more than core defects. Project has become the de facto frontend for serious diffusion work in 2024-2026.

## Alternatives

- `AUTOMATIC1111/stable-diffusion-webui` — use for simpler workflows or SD1.5/SD2-era models.
- InvokeAI — use when you want polished canvas + inpainting workflow with a less intimidating UI.
- Hugging Face `diffusers` library — use for code-first programmatic access.
- Forge / SD.Next — use for AUTO1111-compatible UI with performance tuning.

## Notes

The node-graph paradigm has a steeper learning curve than chat-style UIs but rewards investment for advanced workflows. The "workflow JSON embedded in PNG metadata" feature makes ComfyUI generations highly reproducible — drag any community-shared ComfyUI image into the editor and the full pipeline restores. GPL-3.0 + the active maintainer organization make this both license-clean for personal use and a recurring topic of discussion for commercial pipeline builders.

## Tags

artificial-intelligence, machine-learning, stable-diffusion, image-generation, python, pytorch, node-graph, gpl, diffusion, user-interface
