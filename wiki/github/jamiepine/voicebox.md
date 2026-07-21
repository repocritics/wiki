# jamiepine/voicebox

A local-first desktop voice studio for voice cloning, text-to-speech, dictation, and giving AI agents a voice.

## What it is

Voicebox is an open-source, local-first AI voice application that runs the full voice input/output stack on your own machine. It clones voices from short reference samples, generates speech across multiple TTS engines and languages, transcribes speech to text with Whisper, and lets any MCP-aware agent speak back in a chosen voice. The README positions it as a free, local alternative that combines the output side (ElevenLabs-style TTS) and the input side (WisprFlow-style dictation) in one app for privacy-conscious individual users.

## Key features

- Seven switchable TTS engines (Qwen3-TTS, Qwen CustomVoice, LuxTTS, Chatterbox Multilingual, Chatterbox Turbo, HumeAI TADA, Kokoro) with zero-shot voice cloning and 50+ preset voices.
- Global dictation hotkey with push-to-talk and toggle modes, plus accessibility-verified auto-paste on macOS and an in-app mic on every text field.
- A built-in MCP server and REST API exposing `voicebox.speak`, `voicebox.transcribe`, `list_captures`, and `list_profiles` for agents and scripts.
- Post-processing effects (pitch shift, reverb, delay, chorus, compression, filters) via Spotify's Pedalboard, with reusable presets.
- Unlimited-length generation via sentence-boundary chunking and crossfade, plus a multi-track Stories editor.
- Per-profile voice personalities driven by a bundled local Qwen3 LLM shared across TTS, STT, and dictation refinement.

## Tech stack

- Desktop shell: Tauri (Rust), not Electron; a native Rust shim handles global hotkey, paste injection, and focus introspection.
- Frontend: React, TypeScript, Tailwind CSS, with Zustand and React Query for state; WaveSurfer.js for audio.
- Backend: Python FastAPI (served on port 17493), SQLAlchemy, SQLite, librosa, soundfile.
- Inference: MLX on Apple Silicon, PyTorch (CUDA/ROCm/XPU/DirectML/CPU) elsewhere; Whisper and Whisper Turbo for STT.
- MCP: FastMCP mounted at `/mcp` (Streamable HTTP) plus a bundled stdio shim binary.
- Tooling: Bun (>=1.0, pinned bun@1.3.8) workspaces, Biome, TypeScript ^5.6, Tailwind ^4; the `just` task runner for setup and builds.

## When to reach for it

- You want to clone voices and generate speech locally without sending audio to a cloud service.
- You want system-wide dictation into any application, particularly on macOS where auto-paste is supported.
- You want your MCP-aware coding agent (Claude Code, Cursor, Cline, Windsurf) to speak results back to you.
- You are producing multi-voice conversations, podcasts, or narratives and need a timeline editor.

## When *not* to reach for it

- You want dictation auto-paste on Windows or Linux — those are on the roadmap, not shipped.
- You need a pre-built Linux binary; only build-from-source is available for Linux.
- You have no capable GPU and need fast generation; CPU inference works but is slower.
- You prefer a hosted API with no local model downloads or desktop app to manage.

## Maturity signal

Created in late January 2026 and pushed within a day of this writing, Voicebox is under active, rapid development with a large audience. The high open-issue count and pre-1.0 app version (0.5.0) reflect a young project still filling in platform parity (Windows/Linux paste, more STT engines) as documented in its roadmap. MIT licensing and a detailed living `PROJECT_STATUS.md` indicate an open, actively steered effort.

## Alternatives

- ElevenLabs — use it when you want a hosted TTS service with no local hardware requirement and do not need on-device privacy.
- WisprFlow — use it when you only need dictation and prefer a managed product over a local app.
- Piper or Coqui TTS — use these when you want a library-level TTS engine to embed rather than a full desktop studio.

## Notes

One bundled local LLM, one model cache, and one GPU-memory footprint back three different features — dictation cleanup, voice personalities, and agent-side rewriting — which the README highlights as a deliberate way to avoid loading multiple models. The same on-screen "pill" overlay surfaces both dictation and agent-speech states, so input and output share one mental model.

## Tags

typescript, rust, python, tauri, text-to-speech, speech-to-text, voice-cloning, desktop-app, mcp, fastapi, local-first, machine-learning
