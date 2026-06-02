# upscayl/upscayl

Upscayl — a free, AI-powered image upscaler. Desktop app for Windows / macOS / Linux that runs upscaling models locally.

## What it is

An Electron desktop app that wraps Real-ESRGAN and similar image-upscaling models with a polished GUI. Runs locally on the user's GPU — no cloud upload required. Free to use, donation-supported.

## Key features

- Multiple upscaling models (Real-ESRGAN, Remacri, Ultramix, etc.).
- Batch processing.
- Custom output formats + sizes.
- Local-only inference (no cloud).
- Cross-platform desktop (Windows, macOS, Linux).
- AGPL-3.0 licensed (Free version) + Pro paid version.

## Tech stack

- TypeScript primary.
- Electron.
- Real-ESRGAN-ncnn-vulkan backend.

## When to reach for it

- You're upscaling images and want a privacy-friendly local tool.
- You want batch processing rather than a per-image online service.

## When *not* to reach for it

- You're on a low-end machine — upscaling models need a capable GPU.
- You only have a few images and don't mind cloud services.

## Maturity signal

Actively maintained.

## Alternatives

- waifu2x — older model with simpler UIs.
- Topaz Gigapixel AI — commercial.
- AUTOMATIC1111 with upscaling extension — for SD-flavored use.

## Tags

typescript, electron, image, upscaling, ai, agpl, desktop, cross-platform
