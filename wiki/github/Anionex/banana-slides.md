# Anionex/banana-slides

An AI-native application that generates editable presentations from a prompt, outline, or per-page description using the nano banana pro image model.

## What it is

Banana Slides is a self-hostable web app for producing slide decks without manual layout work. It targets people who need attractive presentations quickly — the README names beginners, PPT professionals looking for layout inspiration, educators, students, and business users. Rather than filling preset templates, it generates each page as an image with the nano banana pro model and lets the user revise it through natural-language instructions. It ships as a Docker Compose stack (frontend plus backend) and is also offered as an online demo and a desktop build.

## Key features

- Three starting points for a deck — a single idea/sentence, an outline, or per-page descriptions — with Markdown import that previews the recognized page count before appending.
- Material parsing: uploaded PDF/DOCX/MD/TXT files are parsed in the background, and key points, image links, and chart info are extracted into the project's material library.
- Conversational ("Vibe") editing: region-level redraws and full-page regeneration via spoken instructions, plus an optional quality-control mode that checks generated images for garbled text and prompt drift before saving a version.
- Template control with single- or multi-template modes: upload images or PDFs to build a template library that the AI parses and matches per page.
- Export to standard PPTX or PDF, a Beta editable-PPTX export, and one-click narrated explainer videos (MP4) with AI voice and subtitles.
- Multiple AI provider formats (Gemini, OpenAI, Volcengine, Vertex, LazyLLM) configurable by environment variable or in-app settings.

## Tech stack

- Frontend: React 18, TypeScript, Vite 5, Zustand.
- Backend: Python 3.10+, Flask 3.0, SQLite, with uv for dependency management and Alembic for migrations.
- Node.js >= 18 for the frontend; backend dependencies include google-genai, openai, anthropic, python-pptx, PyMuPDF, reportlab, edge-tts, elevenlabs, and lazyllm (with dashscope, zhipuai, and volcengine SDKs for domestic providers).
- FFmpeg (with libass) required for explainer-video export; optional LibreOffice for uploading PPTX in the "PPT refresh" feature.
- Packaged and deployed with Docker / Docker Compose, with prebuilt images published to Docker Hub.
- AGPL-3.0 licensed; current version 0.9.0 (desktop RC2).

## When to reach for it

- Turning an idea, outline, or document into a first-draft deck in minutes without arranging layout by hand.
- Producing image-based slides in a consistent visual style from an uploaded reference image or template.
- Iterating on specific slide regions by describing the change instead of editing in a slide editor.
- Generating a narrated explainer video from an existing deck.
- Self-hosting a slide generator behind your own API keys via Docker.

## When *not* to reach for it

- The core rendering depends on nano banana pro and other paid image-generation APIs; the README warns the Google model is comparatively expensive, so high-volume use carries real per-image cost.
- Fully editable PPTX output is still Beta; if you need precise, layered, text-editable slides today, image-based pages may not fit.
- It is a running web service (frontend, backend, SQLite, FFmpeg), not a library you embed — overkill if you only need programmatic PPTX assembly.
- AGPL-3.0 licensing may not suit closed-source commercial use; the author states non-commercial use is free and asks that commercial intent be discussed directly.

## Maturity signal

Created in late November 2025 and pushed within days of this writing, the project is under active, fast-moving development — the changelog lists frequent feature drops through mid-2026, and the version sits at a 0.9.0 release candidate. High star and fork counts, plus Trendshift and HelloGitHub features, indicate strong current attention. The pre-1.0 versioning and several features still marked in progress (multi-layer editable PPTX, web search, agent mode) mean interfaces and behavior are still settling.

## Alternatives

- Gamma — use it instead when you want a fully hosted, no-setup AI deck generator and do not need self-hosting or your own image model.
- python-pptx — use it instead when you need deterministic, programmatic PPTX assembly from code rather than AI-generated image pages.
- NotebookLM slide decks — the README's own comparison positions this project against it; reach for NotebookLM when you want a Google-integrated tool and can accept a page cap and non-editable output.

## Notes

- Slides are produced as generated images rather than native shapes, which is why the editable-PPTX path is a separate Beta effort and why an optional quality-control pass checks for garbled text before saving a version.
- The backend explicitly bundles domestic Chinese provider SDKs (volcengine-python-sdk[ark], dashscope, zhipuai) because the LazyLLM "online-advanced" install group may not publish as a standard PyPI extra.
- The desktop build has no project-root .env; API configuration is done in-app, and data lives in user-selected data/uploads/exports directories that the app does not auto-migrate.

## Tags

typescript, python, react, flask, llm, text-to-image, ppt-generator, slides, self-hosted, docker
