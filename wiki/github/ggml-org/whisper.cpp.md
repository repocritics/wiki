# ggml-org/whisper.cpp

> A dependency-free C/C++ port of OpenAI's Whisper speech-recognition model, tuned to run offline on CPUs and consumer GPUs.

[GitHub repo](https://github.com/ggml-org/whisper.cpp) ·
[License: MIT](https://github.com/ggml-org/whisper.cpp/blob/master/LICENSE)

## Overview

whisper.cpp is Georgi Gerganov's reimplementation of OpenAI's Whisper automatic
speech recognition (ASR) model in plain C/C++[^1]. It does not depend on Python,
PyTorch, or any ML runtime: the model weights are converted to a custom `ggml`
binary format and executed by the `ggml` tensor library — the same foundation
that powers llama.cpp. The stated goals are zero runtime memory allocation,
first-class Apple Silicon support, and easy embedding into other applications.
The repository moved from the personal `ggerganov` account to the `ggml-org`
organization; old URLs redirect[^2].

The project is **inference-only**. It runs existing Whisper checkpoints (tiny
through large-v3, plus large-v3-turbo) but does not train or fine-tune. Every
accuracy characteristic of the underlying model — strong English transcription,
weaker performance on some non-English languages, hallucinated text on silence
or music, approximate word timestamps — is inherited from OpenAI's Whisper, not
introduced or fixed by this port. What whisper.cpp changes is the *deployment
envelope*: it turns a GPU-and-Python model into something that runs on a phone,
a Raspberry Pi, or a CPU-only server. The tradeoff is feature velocity —
supporting a dozen backends with no dependencies keeps the surface small, but
real-world pipeline needs (diarization, robust word alignment, true streaming)
are left as examples or downstream projects rather than supported core features.

## Getting Started

```bash
git clone https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp

# download a model in ggml format
sh ./models/download-ggml-model.sh base.en

# build the CLI
cmake -B build
cmake --build build -j --config Release

# transcribe (input must be 16 kHz mono 16-bit WAV)
./build/bin/whisper-cli -m models/ggml-base.en.bin -f samples/jfk.wav
```

The CLI accepts only 16-bit PCM WAV by default (audio is decoded with the
bundled miniaudio library). Convert other formats first:

```bash
ffmpeg -i input.mp3 -ar 16000 -ac 1 -c:a pcm_s16le output.wav
```

Building with `-DWHISPER_COMMON_FFMPEG=yes` links FFmpeg for broader input
support in the examples.

## Architecture / How It Works

Whisper is an encoder–decoder Transformer. The 30-second audio window becomes a
log-mel spectrogram, the **encoder** produces an audio embedding, and the
**decoder** autoregressively emits text tokens. whisper.cpp implements both
stages as `ggml` compute graphs. The high-level model lives in `src/whisper.cpp`
and the public C API in `include/whisper.h`; everything below that — tensor ops,
backends, quantization — is `ggml`, vendored from the sibling `ggml-org/ggml`
repository.

Backends are selected at build time via CMake flags and dispatched by `ggml` at
runtime:

- **CPU** — AVX/AVX2 on x86, ARM NEON + Apple Accelerate on ARM, VSX on POWER;
  optional OpenBLAS for the encoder's matrix multiplies.
- **Metal** — default GPU path on Apple Silicon; the whole graph runs on-GPU.
- **CUDA** — cuBLAS plus custom kernels for NVIDIA.
- **Vulkan / ROCm (HIP) / MUSA / CANN** — cross-vendor and vendor-specific GPU
  paths.
- **Core ML** and **OpenVINO** — accelerate *only the encoder* on the Apple
  Neural Engine or Intel devices; the decoder still runs through `ggml`. Both
  require a one-time conversion step and a slow first run while the platform
  compiles a device-specific artifact[^3].

Models can be **quantized** (e.g. Q5_0, Q8_0) to shrink disk and memory
footprint at a small accuracy cost, using the bundled `quantize` tool. The
repository also ships example binaries beyond the CLI: `whisper-server` (an
HTTP server with an OpenAI-compatible endpoint), `whisper-stream` (sliding-
window real-time capture via SDL2), and `whisper-bench`. **Voice Activity
Detection (VAD)** was added to gate the decoder on speech regions, which reduces
hallucination on silent or non-speech audio.

## Production Notes

**Hallucination on non-speech.** The most common surprise. On silence, music, or
long pauses, the decoder emits plausible-sounding but invented text, or loops a
repeated phrase — this is a Whisper model behavior, not a bug in the port.
Enabling VAD, trimming silence, and constraining segment length mitigate it but
do not eliminate it.

**Timestamps are approximate.** Segment timestamps are usable; token/word-level
timestamps are derived heuristically (DTW-based) and drift, especially at speech
boundaries. Pipelines needing accurate word alignment typically post-process
with a forced aligner or use WhisperX downstream.

**"Real-time" is a sliding window.** `whisper-stream` re-transcribes overlapping
chunks; it is not incremental streaming. Latency and accuracy trade against
chunk size, and word boundaries at chunk edges are unreliable. Treat it as a
demo pattern, not a production transcription service.

**Encoder offload is partial.** Core ML / OpenVINO only accelerate the encoder.
The autoregressive decoder — often the latency bottleneck on long audio — stays
on `ggml`. Budget the first inference on a device as slow because of one-time
model compilation and caching.

**Model size vs. hardware.** Memory ranges from ~273 MB (tiny) to ~3.9 GB
(large) per the project's own table, before quantization. `large-v3-turbo`
trades a little accuracy for markedly faster decoding and is the usual default
when latency matters. On CPU-only servers, thread count (`-t`) and BLAS are the
main throughput levers. The C API in `whisper.h` is stable and widely bound
(Java, Go, Node/npm `whisper.cpp`, Conan `whisper-cpp`, .NET), but the `ggml`
internals underneath change frequently — expect churn if you vendor `ggml`.

## When to Use / When Not

**Use when:**
- You need offline, on-device, or air-gapped transcription with no Python
  runtime.
- You're embedding ASR into a C/C++, mobile (iOS/Android), or WebAssembly app.
- You want a small, auditable, single-binary dependency you can ship.
- You're targeting Apple Silicon and want Metal/ANE acceleration out of the box.

**Avoid when:**
- You need speaker diarization, robust word-level alignment, or true streaming
  as supported features rather than examples.
- You're on NVIDIA at scale and want maximum throughput — CTranslate2-based
  runtimes are usually faster.
- You need training, fine-tuning, or model modification (inference only).
- Your accuracy requirements exceed what the Whisper checkpoints deliver for
  your language or audio conditions — no runtime fixes that.

## Alternatives

- openai/whisper — the reference PyTorch implementation; easier to hack and
  fine-tune, but Python + GPU heavy. Use when you need training or the canonical
  model, not lean deployment.
- SYSTRAN/faster-whisper — CTranslate2 backend; typically faster and lower-
  memory on NVIDIA than whisper.cpp. Use when throughput on GPUs is the priority
  and Python is acceptable.
- m-bain/whisperX — adds accurate word-level timestamps and speaker diarization
  on top of faster-whisper. Use when alignment and "who spoke when" matter.
- alphacep/vosk-api — Kaldi-based offline ASR with genuine streaming and tiny
  models. Use when you need low-latency streaming on constrained devices and can
  accept lower accuracy than Whisper.
- ggml-org/llama.cpp — sibling project sharing the `ggml` core, for LLM (not
  ASR) inference. Relevant if you're building the same offline stack for text
  generation.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial port | 2022-09 | First public C/C++ port of Whisper on `ggml`[^1]. |
| large-v3 support | 2023-11 | Added after OpenAI released the large-v3 checkpoint. |
| large-v3-turbo | 2024-10 | Faster turbo decoder variant supported. |
| VAD integration | 2025 | Voice Activity Detection to gate the decoder. |
| v1.9.1 | 2026-06-19 | Latest stable release at time of writing (fetched). |

Metadata (fetched): 51,794 stars · 5,913 forks · MIT · C++ · last pushed
2026-07-11.

## References

[^1]: whisper.cpp README, "High-performance inference of OpenAI's Whisper". https://github.com/ggml-org/whisper.cpp
[^2]: Repository resides under the `ggml-org` organization; requests to `github.com/ggerganov/whisper.cpp` redirect. Verified via GitHub API `full_name: ggml-org/whisper.cpp`.
[^3]: whisper.cpp README, "Core ML support" and "OpenVINO support" sections (encoder-only acceleration; first-run compilation). https://github.com/ggml-org/whisper.cpp#core-ml-support

## Tags

c-plus-plus, speech-recognition, speech-to-text, whisper, asr, on-device, ggml, inference, transformer, offline, cross-platform
