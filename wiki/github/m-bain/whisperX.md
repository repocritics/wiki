# m-bain/whisperX

> OpenAI Whisper wrapped with VAD batching, wav2vec2 forced alignment, and pyannote diarization to get fast, word-level, speaker-labeled transcripts.

[GitHub repo](https://github.com/m-bain/whisperX) ·
[Paper (arXiv:2303.00747)](https://arxiv.org/abs/2303.00747) ·
[License: BSD-2-Clause](https://github.com/m-bain/whisperX/blob/main/LICENSE)

## Overview

WhisperX is a research-originated pipeline from Max Bain and the Oxford Visual Geometry Group, published at INTERSPEECH 2023[^1]. It exists to fix three concrete shortcomings of OpenAI's Whisper: utterance-level timestamps that can drift by several seconds, no native batching, and no speaker attribution. It does not train a new ASR model. Instead it orchestrates existing models — Whisper (via the faster-whisper / CTranslate2 backend) for transcription, a voice-activity-detection model to cut audio into speech segments, a phoneme wav2vec2 model for forced alignment, and pyannote-audio for diarization — into a four-stage chain.

The payoff is real: roughly 70x realtime on large-v2 with batched inference, word-level timestamps accurate enough for subtitling, and per-word speaker IDs. With ~23k stars it is the default open-source answer to "I need accurate word timings and who-said-what," widely used for captioning, podcast/meeting tooling, and dataset construction[^2].

The defining tension is that WhisperX is glue, not a monolith. Its quality and its failure modes are inherited from four independently-versioned upstreams (Whisper weights, faster-whisper, torchaudio/HF wav2vec2 pipelines, pyannote). That makes it capable but brittle: a good transcript depends on a language-specific alignment model existing, a VAD that segments cleanly, and a diarization model whose licensing and gated-download flow you have accepted. When one stage is weak, the whole output degrades in ways that are not obvious from the final JSON.

## Getting Started

```bash
pip install whisperx          # PyPI; or: uvx whisperx
# GPU path expects CUDA toolkit 12.8 installed first
```

```python
import whisperx

device, audio_file, batch_size = "cuda", "audio.mp3", 16
model = whisperx.load_model("large-v2", device, compute_type="float16")
audio = whisperx.load_audio(audio_file)
result = model.transcribe(audio, batch_size=batch_size)   # 1. transcribe (batched)

# 2. word-level forced alignment (language-specific model)
model_a, meta = whisperx.load_align_model(language_code=result["language"], device=device)
result = whisperx.align(result["segments"], model_a, meta, audio, device,
                        return_char_alignments=False)

# 3. diarization (needs an accepted HF token for the pyannote model)
from whisperx.diarize import DiarizationPipeline, assign_word_speakers
diarize = DiarizationPipeline(token=HF_TOKEN, device=device)(audio)
result = assign_word_speakers(diarize, result)
```

CPU / macOS: `whisperx audio.wav --compute_type int8 --device cpu`. The CLI mirrors the Python API and writes SRT/VTT/JSON/TSV.

## Architecture / How It Works

The pipeline is four sequential stages, each loading and (ideally) unloading its own model:

1. **VAD segmentation.** Audio is first cut into speech regions by a voice-activity model (pyannote-based by default, with silero-vad as an option). This is what enables batching without the buffered/sliding-window logic of vanilla Whisper, and the paper argues it also reduces word error rate and hallucination on long-form audio[^1].
2. **Batched transcription.** Speech segments are fed to Whisper through the faster-whisper backend (CTranslate2), which is where the large speedup and the sub-8GB VRAM footprint for large-v2 come from. Critically, WhisperX runs Whisper with `without_timestamps=True` (one forward pass per batch item) and `condition_on_prev_text=False` by default — so its raw text can differ from stock Whisper output.
3. **Forced alignment.** Whisper's coarse segment timestamps are discarded and re-derived by aligning the transcript against the audio with a phoneme wav2vec2 model, producing per-word (and optionally per-character) start/end times. This stage is language-specific: default torchaudio pipelines cover en/fr/de/es/it, with many more via Hugging Face models listed in `alignment.py`.
4. **Diarization + assignment.** pyannote produces speaker turns over the whole audio, then `assign_word_speakers` maps each aligned word to a speaker by temporal overlap.

The coupling story matters: alignment cannot time a word the phoneme dictionary can't represent (numerals, currency, symbols like "£13.60" get no timestamp), diarization is a separate model with its own accuracy ceiling, and speaker assignment is a post-hoc overlap join rather than a jointly-trained system. Each `load_*` returns a model you are expected to `del` and `gc.collect()` between stages to fit on modest GPUs.

## Production Notes

- **Diarization is gated and moves.** It requires a Hugging Face token plus manual acceptance of the pyannote model user agreement; recent versions target the `speaker-diarization-community-1` model. Older tutorials and pinned code referencing `pyannote/speaker-diarization-3.1` or the `use_auth_token` argument will break — the diarization dependency has been the single most common source of "worked last month, fails now" issues.
- **Dependency stack is heavy and version-sensitive.** WhisperX sits on faster-whisper, CTranslate2, torch, torchaudio, and pyannote simultaneously. CUDA/cuDNN mismatches (CTranslate2 expecting a specific cuDNN) and torch/torchaudio version skew are frequent install failures; the project standardized on CUDA 12.8. Pin your whole environment, not just whisperx.
- **Diarization quality is the weakest link.** The README itself states diarization "is far from perfect" and overlapping speech is handled poorly by both Whisper and WhisperX. Expect speaker-count and boundary errors on crosstalk, phone audio, and >4 speakers; pass `--min_speakers`/`--max_speakers` when known.
- **Batching changes the text.** Because transcription runs without timestamps and without prior-text conditioning, output can diverge from stock Whisper — usually fewer hallucinations, but do not assume byte-identical transcripts when migrating.
- **Memory management is manual.** The three-model chain will OOM a small GPU if you don't flush between stages. Levers: smaller `--batch_size`, smaller `--model`, `--compute_type int8` (the last two trade accuracy).
- **Alignment coverage is uneven.** For a language outside the default set you must locate and validate a phoneme wav2vec2 model yourself; a wrong or low-quality alignment model silently produces bad word timings rather than an error.

## When to Use / When Not

**Use when:**
- You need word-level timestamps for subtitles, karaoke-style highlighting, or clip extraction.
- You need speaker-labeled transcripts and can accept imperfect diarization.
- You're batch-processing long-form audio and want throughput close to faster-whisper with better timing.

**Avoid when:**
- You only need a plain transcript — faster-whisper or whisper.cpp alone is simpler and lighter.
- You need production-grade, warranted diarization with real speaker names — a managed meeting API or a dedicated diarization system will beat the pyannote overlap-join approach.
- You want a stable, low-maintenance dependency — the multi-upstream, gated-model surface demands active environment care.
- You're on a constrained edge device — the full stack is GPU-hungry and dependency-heavy.

## Alternatives

- SYSTRAN/faster-whisper — the transcription backend WhisperX itself uses; pick it when you want speed but not alignment or diarization.
- openai/whisper — the reference model; use it for simplicity or when you need the exact original decoding behavior.
- ggml-org/whisper.cpp — C/C++ CPU-first inference; use it for local, GPU-free, or embedded transcription.
- pyannote/pyannote-audio — the diarization engine; use it directly when speaker segmentation is the primary goal, not ASR.
- speaches-ai/speaches (formerly faster-whisper-server) — use it when you want an OpenAI-compatible transcription HTTP server rather than a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2022-12 | Repo created; forced-alignment on Whisper for word-level timestamps[^3]. |
| v2 | 2023 | VAD filtering on by default, code cleanup, imports whisper lib. |
| Paper | 2023-03 | arXiv:2303.00747 preprint; batched inference, 60–70x realtime[^1]. |
| v3 | 2023 | faster-whisper backend, 70x speedup open-sourced, sentence-level segments (nltk). |
| INTERSPEECH | 2023 | Paper accepted; 1st place Ego4d transcription challenge[^1]. |
| Ongoing | 2024–2026 | uv-based install, CUDA 12.8, migration to pyannote community-1 diarization[^2]. |

## References

[^1]: Bain, Huh, Han, Zisserman, "WhisperX: Time-Accurate Speech Transcription of Long-Form Audio," INTERSPEECH 2023. https://arxiv.org/abs/2303.00747
[^2]: m-bain/whisperX README and repository. https://github.com/m-bain/whisperX
[^3]: Repository metadata, created 2022-12-09 (GitHub API).

## Tags

python, speech-to-text, asr, whisper, speech-recognition, forced-alignment, speaker-diarization, word-level-timestamps, faster-whisper, pyannote, subtitles
