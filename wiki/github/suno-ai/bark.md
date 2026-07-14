# suno-ai/bark

> A GPT-style text-to-audio model that generates speech, music, and sound effects directly from text — research code that became a de facto TTS baseline, then stopped being maintained.

[GitHub repo](https://github.com/suno-ai/bark) ·
[License: MIT](https://github.com/suno-ai/bark/blob/main/LICENSE)

## Overview

Bark is a transformer-based text-to-audio model released by Suno in April 2023[^1]. Unlike a conventional TTS pipeline (text → phonemes → acoustic features → vocoder), Bark is fully generative: it maps text directly to audio tokens with a GPT-style autoregressive stack, with no phoneme intermediate. This lets it produce things a phoneme pipeline structurally cannot — laughter, sighs, music, ambient noise, hesitations, code-switched multilingual speech — from the same model and the same `[bracket]` prompt vocabulary[^2].

The defining tension is right there in the design. Because Bark is generative rather than deterministic, it "takes creative liberties": the same prompt yields different takes, and outputs can deviate from the script in ways a production TTS system is not supposed to. Suno's own README leads with a disclaimer that the model "was developed for research purposes" and "can deviate in unexpected ways"[^2]. Bark is best understood as a research artifact that happens to be genuinely useful, not as a drop-in speech engine.

The second, larger caveat is maintenance. The last commit to this repository was in August 2024[^3]; Suno's product focus moved to its commercial music-generation models. Bark the repo is effectively frozen — issues and PRs accumulate unmerged (hundreds of open issues[^3]) — even as Bark the model lives on through the Hugging Face Transformers integration, which is where most current users actually run it.

## Getting Started

```bash
# NOTE: `pip install bark` installs an UNRELATED package. Do not use it.
pip install git+https://github.com/suno-ai/bark.git
```

```python
from bark import SAMPLE_RATE, generate_audio, preload_models
from scipy.io.wavfile import write as write_wav

preload_models()  # downloads model weights via Hugging Face on first run

text = "Hello, my name is Suno. [laughs] And I like pizza."
audio_array = generate_audio(text, history_prompt="v2/en_speaker_6")

write_wav("out.wav", SAMPLE_RATE, audio_array)
```

Most new integrations use the Transformers path instead, which pins a versioned API and dependency set:

```python
from transformers import AutoProcessor, BarkModel

processor = AutoProcessor.from_pretrained("suno/bark")
model = BarkModel.from_pretrained("suno/bark")
inputs = processor("Hello, my dog is cute", voice_preset="v2/en_speaker_6")
audio = model.generate(**inputs).cpu().numpy().squeeze()
```

## Architecture / How It Works

Bark follows the AudioLM / VALL-E lineage[^4]: text is converted to discrete audio tokens through a cascade of transformers, and a neural codec turns those tokens back into a waveform. There are three autoregressive stages:

1. **Text → semantic tokens.** A GPT-style model maps the input text to a sequence of coarse "semantic" tokens that capture content and prosody. Language is inferred automatically from the text — there is no language flag — so a German prompt with English words produces English audio with a German accent[^2].
2. **Semantic → coarse acoustic tokens.** A second transformer predicts the first codebooks of the audio representation, conditioned on the semantic tokens (and, when supplied, a voice preset).
3. **Coarse → fine acoustic tokens.** A third model fills in the remaining codebooks.

The final tokens are decoded to a 24 kHz waveform by EnCodec, Meta's neural audio codec[^5]. The GPT stages themselves are architecturally close to nanoGPT, which the README credits directly[^2].

**Voice presets** are pre-computed history token sequences (`v2/en_speaker_0` … `v2/<lang>_speaker_N`, 100+ across supported languages). Passing `history_prompt` biases stages 1–2 toward that speaker's tone, pitch, and prosody. This is not voice cloning — there is no mechanism to enroll an arbitrary target voice, and the README states custom cloning is unsupported[^2]. Special tokens like `[laughter]`, `[MAN]`, `[WOMAN]`, `♪` for lyrics, and CAPITALIZATION for emphasis are prompt conventions, not guaranteed controls.

The context window is the hard architectural constraint that shapes everything downstream: a single `generate_audio` call produces roughly 13–14 seconds of audio. Longer output is done by chaining generations and feeding prior context forward, which the repo documents in a long-form notebook rather than in the core API[^2].

## Production Notes

**Nondeterminism is the primary footgun.** Bark is autoregressive and sampled, so repeated calls diverge and the model can hallucinate extra words, drop text, switch apparent speaker mid-clip, or render speech as music. For anything user-facing you need generate-and-verify loops (e.g. ASR round-tripping the output against the input) and human review — you cannot treat a Bark call as a pure function from text to speech.

**The ~13-second ceiling forces stitching.** Real applications need chunking, per-chunk voice-preset carry-over for consistency, and crossfade/silence handling at the seams. Voice consistency across chunks is imperfect even with a fixed preset.

**VRAM and speed.** The full model needs roughly 12 GB of VRAM to hold all stages on GPU simultaneously[^2]. `SUNO_USE_SMALL_MODELS=True` fits in about 8 GB at slightly lower quality; `SUNO_OFFLOAD_CPU=True` combined with small models runs on cards down to ~2 GB but is much slower. On a strong GPU generation approaches real-time; on CPU or older/colab GPUs it is far slower, which makes CPU-only serving impractical for interactive use.

**Weights and caching.** Model checkpoints download through the Hugging Face hub on first `preload_models()`; control the location with `HF_HOME` / standard HF env vars rather than any Bark-specific setting[^2]. There is no built-in server, batching, or streaming — you get a function that returns a NumPy array.

**Prefer the Transformers integration for production.** Because the source repo is unmaintained[^3], the `transformers` `BarkModel` path is the practical choice: it has a stable versioned API, receives dependency and security updates through the Transformers release cycle, and documents batching and half-precision/`bettertransformer` speedups the standalone repo does not.

**Output fidelity is inherently variable.** The README itself notes outputs can sound like "a 1980s phone call" — the model generates audio from scratch and does not target studio quality[^2]. Do not promise consistent broadcast-grade audio.

## When to Use / When Not

**Use when:**
- You need expressive non-speech audio (laughs, sighs, singing, ambient) that phoneme-based TTS cannot produce.
- You want quick multilingual or code-switched prototypes without wiring up per-language pipelines.
- You are doing research, creative/offline content, or demos where variance is acceptable or even desirable.
- You can tolerate offline batch generation with a human or ASR verification step.

**Avoid when:**
- You need deterministic, script-faithful speech (IVR, accessibility screen-reading, navigation prompts).
- You need custom voice cloning of a specific target speaker.
- You need low-latency streaming or on-device/CPU real-time synthesis.
- You need a maintained upstream with security patches and responsive issues — depend on the Transformers integration instead of this repo.

## Alternatives

- coqui-ai/TTS — broad multi-model TTS toolkit with XTTS voice cloning; use when you need controllable, cloneable, more deterministic speech (note: Coqui the company shut down, but the library remains widely forked).
- rhasspy/piper — fast, fully local neural TTS optimized for CPU and embedded; use when you need real-time, deterministic, low-resource speech.
- fishaudio/fish-speech — actively maintained multilingual TTS with voice cloning; use when you want a modern, maintained alternative with cloning.
- myshell-ai/OpenVoice — instant voice cloning and cross-lingual tone control; use when the target is replicating a specific reference voice.
- hexgrad/kokoro — small, high-quality, permissively licensed TTS; use when you want lightweight, consistent speech without Bark's variance.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2023-04-20 | First public release of Bark[^2]. |
| MIT relicense | 2023-05-01 | Relicensed to MIT for commercial use; 2×/10× GPU/CPU speedups; small-model option; low-VRAM support[^2]. |
| Transformers integration | 2023-07 | `BarkModel` added to Hugging Face Transformers 4.31.0[^6]. |
| Last repo commit | 2024-08-19 | Final push to the repository; effectively unmaintained since[^3]. |

## References

[^1]: Suno, "Bark" model repository — created 2023-04-07. https://github.com/suno-ai/bark
[^2]: Bark README (Suno). https://github.com/suno-ai/bark/blob/main/README.md
[^3]: GitHub API metadata for suno-ai/bark: last push 2024-08-19, ~269 open issues, not archived (fetched 2026-07-15). https://github.com/suno-ai/bark
[^4]: VALL-E, "Neural Codec Language Models are Zero-Shot Text to Speech Synthesizers" (arXiv:2301.02111); AudioLM (arXiv:2209.03143).
[^5]: EnCodec — Meta's neural audio codec used for Bark's audio tokens. https://github.com/facebookresearch/encodec
[^6]: Bark in Hugging Face Transformers. https://huggingface.co/docs/transformers/main/en/model_doc/bark

## Tags

python, jupyter-notebook, text-to-speech, text-to-audio, generative-audio, transformer, gpt, multilingual, voice-synthesis, machine-learning, mit-license, unmaintained
