# deezer/spleeter

> Pretrained TensorFlow models that split a song into vocals, drums, bass, and other stems from the command line.

[GitHub repo](https://github.com/deezer/spleeter) ·
[Official website](https://research.deezer.com/projects/spleeter.html) ·
[License: MIT](https://github.com/deezer/spleeter/blob/master/LICENSE)

## Overview

Spleeter is a music source separation library released by Deezer's research team
in November 2019[^1]. It shipped something the field had mostly kept in
academia: pretrained models that a non-researcher could run on a real song in
one command. The 2-, 4-, and 5-stem models — trained on the MUSDB18 dataset[^2]
— made "isolate the vocals" a `pip install` away, and the project became the
default answer to stem separation almost overnight. At ~28k stars it remains one
of the most-starred audio-ML projects on GitHub.

Its defining tension in 2026 is that Spleeter won the popularity contest but lost
the quality race. The architecture is a spectrogram-masking U-Net, and
waveform-domain successors (notably Demucs) now produce audibly cleaner
separations, especially on drums and bass. Spleeter is still fast, still trivial
to script, and still a reasonable baseline — the Music Demixing Challenge used it
as one[^3] — but it is effectively feature-frozen, pinned to an aging TensorFlow
stack, and no longer where the state of the art lives. You use it because it is
easy and fast, not because it is best.

## Getting Started

`ffmpeg` and `libsndfile` are system dependencies and must exist before
separation will run.

```bash
pip install spleeter
# fetch a sample clip
wget https://github.com/deezer/spleeter/raw/master/audio_example.mp3
# 2-stem split: vocals + accompaniment
spleeter separate -p spleeter:2stems -o output audio_example.mp3
# -> output/audio_example/{vocals.wav, accompaniment.wav}
```

```python
from spleeter.separator import Separator

# 4stems = vocals / drums / bass / other; downloads the model on first use
separator = Separator("spleeter:4stems")
separator.separate_to_file("audio_example.mp3", "output/")
```

## Architecture / How It Works

Spleeter is a thin, well-organized wrapper around a set of pretrained
TensorFlow models. The signal path is classic spectrogram masking:

1. Decode audio to a waveform (via `ffmpeg`), resample to 44.1 kHz stereo.
2. Compute the Short-Time Fourier Transform (STFT) and take its magnitude.
3. Feed the magnitude spectrogram to a per-configuration U-Net that estimates a
   soft mask per target stem.
4. Multiply the masks against the original complex STFT and invert (ISTFT) back
   to waveforms.

Each stem count (2/4/5) is a distinct trained model, not a single model with
switchable heads. The models are downloaded on first use and cached locally
(`~/.cache` or the `MODEL_PATH` env var). Because separation is a forward pass
over a spectrogram, it is cheap: Deezer reports ~100x faster than real time on a
GPU, and it stays usable on CPU.

The masking-in-magnitude-domain approach is also the ceiling. Phase is taken
from the original mixture rather than estimated, so overlapping sources leak —
the well-known "watery" or "phasey" artifacts in isolated vocals. Waveform-domain
models sidestep this by operating on samples directly, which is the core reason
they now outperform Spleeter on the same benchmark.

## Production Notes

The differentiator here is that the failure modes are almost all
operational, not algorithmic.

- **Default 10-minute truncation.** Spleeter only processes the first 600
  seconds of a file by default. Longer tracks are silently cut off unless you
  raise the duration (`-d`/`--duration` on the CLI, `duration` in the API). This
  surprises people running it on DJ sets or podcasts.
- **TensorFlow version pinning is the top install footgun.** Spleeter constrains
  its TensorFlow (and therefore its Python) versions tightly. On a machine with
  a newer Python or an existing TF install, resolution fails or pulls an
  incompatible wheel. Install it in a dedicated virtualenv/conda env, never into
  a shared environment.
- **Apple Silicon is rough.** M1/M2 installs fail or misbehave largely because of
  TensorFlow wheel availability on arm64; the README itself links a community
  workaround[^4]. Docker (x86 emulation) is the reliable escape hatch.
- **GPU is no longer a separate package.** Spleeter 2.1.0 dropped the dedicated
  `spleeter-gpu` distribution and renamed CLI input options[^5]; older tutorials
  referencing `spleeter-gpu` or `-i` are stale.
- **Model download at runtime.** First run reaches out to fetch pretrained
  weights. In air-gapped or firewalled CI, pre-seed the model cache and point
  `MODEL_PATH` at it, or the job hangs/fails on network.
- **Memory scales with clip length.** The whole spectrogram is held in memory;
  very long inputs can OOM even within the duration limit. Batch by splitting
  long files.
- **Maintenance is minimal.** Commits still land occasionally, but there have
  been no new models or architectural changes for years, and the open-issue
  count (200+) reflects a triage backlog more than active development. Treat it
  as stable-but-static.

## When to Use / When Not

**Use when:**
- You need fast, scriptable stem separation and "good enough" quality (karaoke
  tracks, pre-processing for MIR, quick demos).
- You want a stable CLI/Python API you can pin and forget.
- You are CPU-bound or need to process large catalogs where speed beats a few dB
  of separation quality.

**Avoid when:**
- Output quality is the priority — Demucs and MDX-based models separate audibly
  better, especially drums/bass and isolated vocals.
- You are on Apple Silicon and want a frictionless install.
- You need active maintenance, newer architectures, or transformer-based
  separators; Spleeter's stack is frozen in the TF 2.x-of-2020 era.

## Alternatives

- facebookresearch/demucs — waveform-domain (Hybrid Transformer) separation with
  clearly higher quality; use when output fidelity matters more than raw speed.
- sigsep/open-unmix-pytorch — the reference open PyTorch baseline; use when you
  want a hackable, research-oriented model to modify or retrain.
- Anjok07/ultimatevocalremovergui — desktop GUI bundling Demucs/MDX-Net models;
  use when you want best-in-class vocal removal without writing code.
- ZFTurbo/Music-Source-Separation-Training — training/inference framework for
  modern architectures; use when you need to train or fine-tune your own separator.
- adefossez/julius — lightweight DSP building blocks (resampling, filtering); use
  when you only need signal-processing primitives, not full separation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019-11 | Initial release: 2/4/5-stem pretrained models, CLI + API[^1]. |
| JOSS paper | 2020-06 | Published in Journal of Open Source Software[^6]. |
| 2.0 | 2020-06 | Migration to TensorFlow 2.x. |
| 2.1.0 | 2020-08 | Dropped `spleeter-gpu` package, renamed CLI input options[^5]. |
| 2.3.x | 2022–2023 | Dependency/compatibility maintenance; no new models. |

## References

[^1]: Deezer Engineering, "Releasing Spleeter: Deezer R&D source separation
engine" — 2019. https://medium.com/deezer-engineering/releasing-spleeter-deezer-r-d-source-separation-engine-2b88985e797e
[^2]: MUSDB18 dataset, SigSep. https://sigsep.github.io/datasets/musdb.html
[^3]: Music Demixing Challenge (ISMIR 2021), AIcrowd. https://www.aicrowd.com/challenges/music-demixing-challenge-ismir-2021
[^4]: Spleeter issue #607 — Apple M1 workaround. https://github.com/deezer/spleeter/issues/607#issuecomment-1021669444
[^5]: Spleeter CHANGELOG — 2.1.0 breaking changes. https://github.com/deezer/spleeter/blob/master/CHANGELOG.md
[^6]: Hennequin et al., "Spleeter: a fast and efficient music source separation
tool with pre-trained models", JOSS 5(50):2154, 2020. https://doi.org/10.21105/joss.02154

## Tags

python, tensorflow, audio-processing, source-separation, music-information-retrieval, deep-learning, pretrained-models, stem-separation, deezer, cli
