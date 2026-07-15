# facebookresearch/demucs

> PyTorch music source separation — split a mixed track into drums, bass, vocals, and other stems with a hybrid spectrogram/waveform model.

[GitHub repo](https://github.com/facebookresearch/demucs) ·
[License: MIT](https://github.com/facebookresearch/demucs/blob/main/LICENSE)

## Overview

Demucs is a music source separation library from Meta's FAIR lab, built by Alexandre Défossez and collaborators. Given a stereo mix it produces separated stems — by default `drums`, `bass`, `vocals`, and `other` — as 44.1 kHz WAV (or MP3/FLAC) files. It is one of the two reference open-source separators, alongside Deezer's Spleeter, and the model most commonly cited when quality matters over speed.

The lineage runs across four generations: v1/v2 were pure waveform-domain U-Nets (Conv-Tasnet-style), v3 introduced Hybrid Demucs — a model operating in both the spectrogram and waveform domains, which won the 2021 Sony Music Demixing (MDX) challenge[^1]. v4 (the current default, `htdemucs`) replaces the innermost layers with a cross-domain Transformer encoder that uses self-attention within each domain and cross-attention across them[^2]. The fine-tuned v4 model reaches 9.0 dB overall SDR on the MUSDB HQ test set per the paper[^2].

The defining fact about this repository as of 2026 is that **it is archived and unmaintained**. Défossez left Meta, and the README directs users to a personal fork at `adefossez/demucs`, which itself receives only critical bug fixes — no feature work, no support for individual use cases[^3]. This repo is now a stable, frozen snapshot: the code and pretrained weights still work, but nothing here will change.

## Getting Started

```bash
python3 -m pip install -U demucs   # Python 3.8+, pulls in torch/torchaudio
```

```bash
# Separate into the 4 default stems (htdemucs), output under separated/htdemucs/<track>/
demucs "my track.mp3"

# Karaoke mode: vocals vs. everything else (still separates fully internally)
demucs --two-stems=vocals "my track.mp3"

# CPU fallback if you run out of GPU memory; ~1.5x track duration to process
demucs -d cpu "my track.mp3"
```

```python
# Programmatic use — pass a parsed CLI argv to the separate entrypoint
import demucs.separate
demucs.separate.main(["--mp3", "--two-stems", "vocals", "-n", "mdx_extra", "track.mp3"])
```

## Architecture / How It Works

The core (`htdemucs`) is a dual U-Net: one branch processes the raw waveform (temporal domain), one processes the STFT spectrogram (spectral domain). Both branches encode down to a bottleneck where a cross-domain Transformer exchanges information, then decode back up. The spectral branch's output is inverse-STFT'd and summed with the temporal branch's output, so the final prediction is genuinely hybrid rather than a post-hoc blend[^2]. This hybrid design is what let v3/v4 pull ahead of purely spectrogram methods (Spleeter, Open-Unmix) on SDR while keeping fewer waveform-domain artifacts.

Inference is chunked. Audio is split into fixed-length segments (the HT Transformer models cap at 7.8 s per segment because of the attention window), processed with overlapping windows (`--overlap`, default 0.25), and reassembled with crossfades. The optional "shift trick" (`--shifts=N`) averages N predictions over random time offsets to reduce artifacts at N times the cost. Output stems are automatically rescaled to avoid clipping, which can shift relative stem volumes unless you pass `--clip-mode clamp`.

Model selection is via `-n`. The relevant weights: `htdemucs` (default v4), `htdemucs_ft` (fine-tuned, ~4x slower, marginally better), `htdemucs_6s` (adds `guitar` and `piano` sources — piano quality is poor by the author's own admission), `hdemucs_mmi` (retrained v3), and the `mdx`/`mdx_extra` challenge models. Weights download on first use and are cached locally. The sparse-attention model from the paper that reaches 9.2 dB SDR is **not** released — it needs custom CUDA kernels that were never shipped[^2].

## Production Notes

- **GPU memory is the main footgun.** The defaults want ~7 GB of VRAM; the model runs in as little as 3 GB if you shrink `--segment` (e.g. `--segment 8`), at some quality cost. `PYTORCH_NO_CUDA_MEMORY_CACHING=1` squeezes it further. Below that, use `-d cpu`, where a 4-minute track takes roughly 6 minutes of wall time.
- **`--two-stems` is not a shortcut.** Karaoke mode still runs full 4-stem separation and then sums the unwanted stems, so it is neither faster nor lighter than a full separation.
- **`-j` (parallel jobs) multiplies RAM.** Each worker holds its own copy of the model and buffers; set it conservatively.
- **Platform audio I/O is uneven.** `torchaudio` handles most formats on Linux/macOS, but Windows support is limited and Demucs falls back to `ffmpeg` — read the per-OS docs before filing anything. Output defaults to int16 WAV; `--float32`/`--int24`/`--mp3` change encoding.
- **Frozen dependency surface.** Because the repo is archived, it is pinned to the PyTorch era it was last tested against. Newer torch/torchaudio releases can and do break installs; expect to pin `torch`/`torchaudio` versions or use the Conda environment files rather than assume a clean `pip install` on a fresh 2026 stack.
- **No support channel.** Bug reports, feature requests, and "doesn't work on my track" issues are explicitly out of scope here; the community has largely migrated to downstream GUIs and forks.

## When to Use / When Not

**Use when:**
- You want the best readily-available open-source separation quality and can tolerate GPU-scale compute.
- You need offline, self-hosted, license-clean (MIT) separation with no API dependency.
- You're building on top of a stable, unchanging model — the archival status is a feature for reproducibility.

**Avoid when:**
- You need real-time or low-latency separation — Demucs is offline and heavy (use a purpose-built realtime plugin).
- You need active maintenance, new models, or vendor support — the frozen state is disqualifying.
- You're on constrained hardware with no GPU and large batch volumes — CPU throughput is slow.
- You want a maintained upstream — track `adefossez/demucs` (bug fixes) or downstream tools instead.

## Alternatives

- adefossez/demucs — the author's personal fork and the live successor; use it for the latest fixes.
- deezer/spleeter — TensorFlow, much faster, lower quality; use when speed and simplicity beat SDR.
- sigsep/open-unmix-pytorch — simpler spectrogram baseline; use when you want a small, readable reference model.
- Anjok07/ultimatevocalremovergui (UVR) — desktop GUI that bundles Demucs and MDX-Net models; use when you want no-code stem extraction.
- CarlGao4/Demucs-Gui — a dedicated GUI wrapper around Demucs; use when you want the Demucs models without the command line.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2019 | Initial waveform-domain U-Net separator. |
| v2 | 2021 | Improved waveform model; 6.3 dB overall SDR baseline[^2]. |
| v3 (Hybrid Demucs) | 2021-11 | Hybrid spectrogram/waveform domain; won Sony MDX 2021[^1]. |
| v3.0.4 | 2022-02 | Two-stem (karaoke) mode; float32/int24 export. |
| v4 (HT Demucs) | 2022-11 | Cross-domain Transformer; `htdemucs` default[^2]. |
| v4 on PyPI | 2022-12 | v4 published to PyPI; 6-source model added. |
| SDX 2023 support | 2023-02 | Tooling for the Sound Demixing Challenge 2023. |
| Archived | ~2024 | Repo frozen; development moves to `adefossez/demucs`[^3]. |

## References

[^1]: Alexandre Défossez, "Hybrid Spectrogram and Waveform Source Separation" — ISMIR 2021 MDX Workshop. https://arxiv.org/abs/2111.03600
[^2]: Simon Rouard, Francisco Massa, Alexandre Défossez, "Hybrid Transformers for Music Source Separation" — ICASSP 2023. https://arxiv.org/abs/2211.08553
[^3]: Demucs README — repository archival notice and fork pointer. https://github.com/facebookresearch/demucs

## Tags

python, pytorch, audio, music-source-separation, deep-learning, transformer, spectrogram, stem-separation, audio-processing, machine-learning, unmaintained
