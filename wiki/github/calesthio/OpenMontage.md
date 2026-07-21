# calesthio/OpenMontage

An open-source system that turns an AI coding assistant into an end-to-end video production pipeline, from research and scripting through asset generation, editing, and final render.

## What it is

OpenMontage is an agent-first video production system: instead of shipping a code orchestrator, it treats your AI coding assistant (Claude Code, Cursor, Copilot, Windsurf, or Codex) as the orchestrator that reads YAML pipeline manifests and Markdown "director" skills to execute each stage. It targets creators and operators who want to produce finished videos — explainers, cinematic trailers, documentary montages, talking heads, and more — by describing them in plain language rather than assembling a video-editing toolchain by hand. A stated distinction is that it can build videos from real motion footage retrieved from free and open archives, not only from animated still images.

## Key features

- Twelve production pipelines (animated explainer, animation, avatar spokesperson, cinematic, clip factory, documentary montage, hybrid, localization/dub, podcast repurpose, screen demo, talking head) that each follow a research → proposal → script → scene_plan → assets → edit → compose flow.
- A scored provider selector that ranks each available tool across seven dimensions (task fit, output quality, control, reliability, cost efficiency, latency, continuity) and logs the decision, with alternatives considered and confidence scores.
- A zero-key path: `make setup` provides Piper TTS, open archival footage (Archive.org, NASA, Wikimedia Commons), Remotion and HyperFrames composition, FFmpeg post-production, and built-in subtitles without paid API keys.
- Documentary-montage pipeline that builds a CLIP-searchable corpus from free/open sources and cuts actual motion footage into a finished timeline.
- Backlot, a local "living storyboard" board server that fills itself in as the pipeline runs and acts as an approval gate for script and assets before rendering.
- Quality gates including pre-compose validation and a mandatory post-render self-review (ffprobe, frame extraction, audio analysis, delivery-promise verification, subtitle checks).

## Tech stack

- Primary language: Python (3.10+ per the README prerequisites).
- Core dependencies (`requirements.txt`): pyyaml>=6.0, pydantic>=2.0, jsonschema>=4.20, python-dotenv>=1.0, Pillow>=10.0, numpy>=1.24, requests>=2.31.
- Provider SDKs: google-auth>=2.0 and google-genai>=1.0.0 (Google TTS/Imagen/Veo/Gemini), openai>=2.44.0 (Videos API / Sora 2).
- Backlot board server: fastapi>=0.110, uvicorn>=0.29, watchfiles>=0.21.
- Node.js 18+ with a Remotion (React) composition engine (`remotion-composer/`) and an HTML/CSS/GSAP "HyperFrames" renderer.
- FFmpeg for encoding, subtitle burn-in, audio mixing, and color grading.
- Optional local GPU video models (WAN 2.1, Hunyuan, CogVideo, LTX-Video) via `make install-gpu`.

## When to reach for it

- You already work inside an AI coding assistant and want it to drive a full, multi-stage video production rather than return a single generated clip.
- You want to produce videos with no paid keys, using free TTS, open archival footage, and React/HTML rendering.
- You need real-footage documentary montages, tone poems, or stock-footage collages assembled from open sources rather than animated stills.
- You want per-decision cost estimation, spend caps, and an auditable log of every provider and creative choice.
- You want to start from a reference video (YouTube, Short, Reel, TikTok, or local clip) and get a grounded, differentiated production plan.

## When *not* to reach for it

- You just want one clip from a single prompt with no pipeline, approvals, or self-review overhead.
- You cannot or do not want to run an AI coding assistant as the orchestrator, since there is no standalone code orchestrator.
- You need a fully hands-off, no-approval automation path; the design deliberately pauses at creative decision points for human sign-off.
- Your environment cannot provide the local prerequisites (Python, Node.js, FFmpeg) or, for local generation, a capable GPU.
- The AGPL-3.0 license is incompatible with how you intend to distribute or host derived software.

## Maturity signal

The repository was created in late March 2026 and last pushed within days of this writing, so it is under active, near-daily development. Its very high star and fork counts accumulated over only a few months, which signals strong early momentum rather than a long, proven track record; the roughly 185 open issues are consistent with a young, fast-moving project still stabilizing. The AGPL-3.0 license is a deliberate copyleft choice that will matter for anyone embedding it in proprietary or hosted products. Treat it as early-stage and actively maintained, with the usual churn that implies.

## Alternatives

- Remotion — reach for it directly when you want to author video compositions in React yourself and do not need the agent-driven pipeline, provider selection, or approval gates layered on top (OpenMontage uses it internally as one renderer).
- MoviePy — use when you need straightforward programmatic clip editing and concatenation in Python without AI generation or multi-stage orchestration.
- Commercial single-prompt video generators (for example Runway or Pika) — use when you want one polished clip from a prompt and are content with a hosted service instead of a self-run, multi-pipeline production system.

## Notes

- The architecture is explicitly three layers: `tools/` + `pipeline_defs/` ("what exists"), `skills/` ("how OpenMontage wants it used"), and `.agents/skills/` ("how the underlying technology works"), so most creative logic and quality standards live in inspectable YAML manifests and Markdown rather than code.
- The README reports finished pieces produced for very low cost (examples cited at $0.02–$1.33), and it addresses agent readers directly with a documented "read the contract first" onboarding path (`AGENT_GUIDE.md`, `PROJECT_CONTEXT.md`).

## Tags

python, video-production, agentic-ai, llm, text-to-video, text-to-speech, image-generation, ffmpeg, remotion, pipeline
