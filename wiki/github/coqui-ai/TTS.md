# coqui-ai/TTS

> A PyTorch text-to-speech toolkit spanning classic spectrogram+vocoder pipelines through XTTS voice cloning — now community-maintained after Coqui's shutdown.

[GitHub repo](https://github.com/coqui-ai/TTS) ·
[Official website](http://coqui.ai) ·
[License: MPL-2.0](https://github.com/coqui-ai/TTS/blob/dev/LICENSE.txt)

## Overview

Coqui TTS is a deep-learning speech-synthesis library that packages a large catalog of published TTS architectures — Tacotron/Tacotron2, Glow-TTS, VITS, FastPitch, plus vocoders like HiFiGAN and MelGAN — behind a single Python API and `tts` CLI. It descends directly from Mozilla TTS: founder Eren Gölge (`erogol`) led that project at Mozilla, and Coqui was the commercial continuation after Mozilla wound the team down[^1]. Its most-used artifact is ⓍTTS (XTTS v2), a multilingual voice-cloning model covering 16+ languages with sub-200 ms streaming latency, plus wrappers that expose ~1100 Fairseq/MMS language models and third-party models (Bark, Tortoise) through the same interface[^2].

The defining fact about this repository in 2026 is that **the company behind it shut down in early 2024**, and the repo has received no commits since[^3]. The last push to `dev` (the default branch) was 2024-08-16. At ~45k stars it remains the most-referenced open TTS toolkit, but the canonical GitHub repo is effectively frozen: issues are closed, PRs are not merged, and dependency drift (PyTorch, `transformers`, NumPy 2.x, Python 3.12+) is now the user's problem. Active maintenance moved to a community fork (see Production Notes).

The other structural tension is licensing. The **code** is MPL-2.0 (permissive, commercial-friendly), but the flagship **XTTS model weights are released under the Coqui Public Model License (CPML), which is non-commercial**[^4]. Teams routinely conflate "the repo is open source" with "I can ship XTTS in a product" — those are different questions with different answers.

## Getting Started

```bash
# Python 3.9–3.11. The original PyPI package:
pip install TTS
# Recommended in 2026 — the maintained community fork (drop-in):
pip install coqui-tts
```

```python
import torch
from TTS.api import TTS

device = "cuda" if torch.cuda.is_available() else "cpu"

# XTTS v2: multilingual, zero-shot voice cloning from a reference clip
tts = TTS("tts_models/multilingual/multi-dataset/xtts_v2").to(device)

tts.tts_to_file(
    text="Hello world!",
    speaker_wav="my/reference_voice.wav",  # 6–20s clean clip
    language="en",
    file_path="output.wav",
)
```

```bash
# CLI — defaults to an English LJSpeech model if none specified
tts --text "Text for TTS" --model_name "tts_models/en/ljspeech/glow-tts" \
    --out_path speech.wav
tts --list_models   # browse the ~1100+ model zoo
```

## Architecture / How It Works

The codebase is organized around three swappable model families plus a training harness:

1. **`TTS/tts/`** — acoustic models mapping text → spectrogram (Tacotron2, Glow-TTS, FastPitch, Overflow) and end-to-end models that go text → waveform directly (VITS, YourTTS, XTTS). Each model is a subclass with its own `config` dataclass.
2. **`TTS/vocoder/`** — neural vocoders (HiFiGAN, MelGAN, ParallelWaveGAN, WaveGrad, UnivNet) converting spectrograms → audio for the two-stage pipelines. End-to-end models bypass this stage.
3. **`TTS/vc/`** and **`TTS/speaker_encoder/`** — voice-conversion (FreeVC) and speaker-embedding models (GE2E) that supply the speaker conditioning used by multi-speaker and cloning models.

Model selection flows through a **model-zoo manifest** (`.models.json`): the `tts_models/<lang>/<dataset>/<model>` naming scheme resolves to a download URL, and `TTS.api.TTS` / `Synthesizer` handle fetching weights, loading config, and running inference. This indirection is why `TTS("tts_models/...")` "just works" but also why a stale manifest or moved download host silently breaks first-run downloads.

Training was factored out into a **separate `coqui-ai/Trainer` package** (`from trainer import Trainer`), so the TTS repo defines models and configs while Trainer owns the loop, logging (TensorBoard), and checkpointing. XTTS itself is a GPT-style autoregressive model that predicts audio codec tokens conditioned on a speaker embedding, decoded through a HiFiGAN-based decoder — architecturally closer to modern LLM-style TTS than the older Tacotron pipelines it shares a repo with.

## Production Notes

**Use the community fork.** The canonical `coqui-ai/TTS` is unmaintained. Idiap Research Institute maintains an active hard fork, `idiap/coqui-ai-TTS`, published to PyPI as **`coqui-tts`**[^5]. It is a drop-in replacement (same `TTS` import path) with fixes for newer Python, PyTorch, and `transformers` versions. Net-new projects should install `coqui-tts`, not `TTS`. This is the single highest-value fact for an operator evaluating this repo.

**The XTTS license is a commercial footgun.** MPL-2.0 covers the code; the XTTS weights are CPML (non-commercial, no derivative distribution without permission). Building a paid product on XTTS v2 is a licensing violation, not merely a gray area. Older models (VITS, Glow-TTS on LJSpeech/VCTK) generally carry permissive dataset-derived licenses and are the commercial-safe path.

**Dependency pinning is mandatory.** The frozen repo pins against a mid-2024 dependency snapshot. Installing plain `TTS` into a fresh 2026 environment frequently fails on NumPy 2.x, `transformers` API drift, or Python 3.12+ (the original supports `>=3.9,<3.12`). Pin exact versions or use the fork.

**Voice-cloning quality is reference-sensitive.** XTTS zero-shot cloning degrades sharply with noisy, reverberant, or very short reference clips. Budget for 6–20 s of clean, single-speaker audio. Cloning arbitrary voices also carries consent/impersonation risk — the model does not gate on speaker identity.

**GPU expectations.** XTTS inference is comfortable on a mid-range GPU; the <200 ms streaming figure assumes CUDA. CPU inference works for the lighter Tacotron/VITS models but is slow for XTTS. Old two-stage pipelines (acoustic + separate vocoder) have more moving parts and more failure surface than the end-to-end VITS/XTTS path.

## When to Use / When Not

**Use when:**
- You want open, self-hosted multilingual voice cloning and can accept XTTS's non-commercial terms (research, internal tooling, personal use).
- You need a broad menu of published TTS architectures to train or fine-tune on your own data.
- You want an offline, no-API-key alternative to cloud TTS for privacy-sensitive workloads.

**Avoid when:**
- You need a commercially licensed cloning model out of the box — XTTS weights forbid it; look elsewhere or use only permissive older models.
- You want a vendor-supported, actively patched library — install the `coqui-tts` fork instead of this repo, or use a maintained alternative.
- You need the lowest-latency, smallest-footprint on-device TTS — Piper is leaner for embedded/edge.

## Alternatives

- rhasspy/piper — fast, lightweight ONNX VITS voices for edge/Raspberry-Pi; use when you need offline low-latency TTS without cloning.
- suno-ai/bark — expressive generative audio (music, effects, voices); use when prosody variety matters more than controllability.
- idiap/coqui-ai-TTS — the maintained fork of this exact codebase; use this instead of `coqui-ai/TTS` for any new work.
- espnet/espnet — research-grade speech toolkit (ASR+TTS); use when you need reproducible papers and training rigor over turnkey inference.
- fishaudio/fish-speech — newer open multilingual cloning model; use when you want a more actively developed cloning stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-05-20 | Forked/continued from Mozilla TTS under Coqui[^1]. |
| 0.x series | 2021–2022 | VITS, YourTTS, FastPitch, HiFiGAN model coverage. |
| XTTS v1 | 2023-09 | Production multilingual cloning model, 13 languages[^2]. |
| XTTS v2 | 2023-11 | 16 languages, quality gains, <200 ms streaming[^2]. |
| Coqui shutdown | 2024-01 | Company wound down; maintenance effectively ends[^3]. |
| last commit | 2024-08-16 | Final push to `dev`; repo dormant thereafter[^3]. |

## References

[^1]: Coqui / Mozilla TTS lineage — Eren Gölge led Mozilla TTS before founding Coqui. https://github.com/mozilla/TTS
[^2]: "Open XTTS" — Coqui blog, XTTS v1/v2 release notes (13→16 languages, streaming). https://coqui.ai/blog/tts/open_xtts
[^3]: Eren Gölge, "Coqui is shutting down" (Jan 2024); repo `pushed_at` 2024-08-16 per GitHub API. https://github.com/coqui-ai/TTS
[^4]: Coqui Public Model License (CPML) — non-commercial terms for XTTS weights. https://coqui.ai/cpml
[^5]: idiap/coqui-ai-TTS — maintained community fork, PyPI `coqui-tts`. https://github.com/idiap/coqui-ai-TTS

## Tags

python, pytorch, text-to-speech, tts, voice-cloning, speech-synthesis, deep-learning, xtts, vocoder, multilingual, unmaintained
