# PaddlePaddle/PaddleOCR

> A multilingual OCR and document-parsing toolkit from Baidu, built on the PaddlePaddle framework, that turns images and PDFs into structured text, Markdown, and JSON.

[GitHub repo](https://github.com/PaddlePaddle/PaddleOCR) ·
[Official website](https://www.paddleocr.com) ·
[License: Apache-2.0](https://github.com/PaddlePaddle/PaddleOCR/blob/main/LICENSE)

## Overview

PaddleOCR is Baidu's open-source OCR toolkit, first released in 2020 and still one of the most-starred OCR projects on GitHub[^1]. Its original claim to fame was **PP-OCR**: a deliberately lightweight three-stage pipeline (text detection → angle classification → text recognition) whose smallest models are only a few megabytes and run acceptably on CPU and mobile hardware[^2]. That "ultra-lightweight" design goal — commercial-grade accuracy at a tiny footprint — is the thread running through the whole project, and it is the main thing distinguishing it from heavier research OCR stacks.

Its strongest suit is Chinese and other CJK text, followed by a broad multilingual reach (the project advertises 100+ languages, though quality varies widely by script and is best for Chinese, English, Japanese, and Latin-script languages). Over time it grew beyond line-level OCR into **PP-Structure** (layout analysis, table recognition, key-information extraction) and, more recently, into end-to-end document parsing that emits LLM-ready Markdown/JSON for RAG pipelines.

The defining tension is the **PaddlePaddle dependency**. PaddleOCR is not a PyTorch project; it is built on Baidu's own deep-learning framework, and installing a working `paddlepaddle` / `paddlepaddle-gpu` build matched to your CUDA version is the single most common source of friction for teams outside the Paddle ecosystem[^3]. Documentation has historically been Chinese-first with machine-translated English lagging behind. You accept those costs in exchange for small, fast, genuinely capable models and permissive Apache-2.0 licensing.

## Getting Started

```bash
pip install paddlepaddle          # CPU; use paddlepaddle-gpu for CUDA (match your CUDA version)
pip install paddleocr
```

```python
from paddleocr import PaddleOCR

# Models download on first run (from Baidu / HuggingFace / ModelScope mirrors).
ocr = PaddleOCR(lang="en")

# 3.x API: .predict(); the long-lived 2.x API used ocr.ocr(img, cls=True).
result = ocr.predict("invoice.jpg")

for res in result:
    res.print()                 # boxes + recognized text + confidence
    res.save_to_json("output/")
    res.save_to_img("output/")  # visualization overlay
```

## Architecture / How It Works

Classic **PP-OCR** is a modular pipeline of three independently trained models[^2]:

1. **Detection** — a segmentation-based detector (DB / Differentiable Binarization and its successors) produces text-region polygons.
2. **Angle classification** — an optional lightweight classifier corrects 0°/180° rotation of cropped text lines.
3. **Recognition** — a CRNN/SVTR-style recognizer transcribes each cropped line into a string with a confidence score.

Because the stages are separate, you can swap a heavier detector with a lighter recognizer, retrain one stage, or drop the angle classifier — the modularity is why the same repo serves both mobile and server deployments.

**PP-Structure** layers document understanding on top: layout detection segments a page into regions (text, title, table, figure), a table-structure model reconstructs cells and grid, and a KIE component extracts key/value fields. This is what turns a scanned form into structured records rather than a bag of lines.

The 3.x line was a substantial reorganization: pipelines were unified under Baidu's **PaddleX** abstraction, and the project added an end-to-end **document vision-language model** (the PaddleOCR-VL series, built on a compact ERNIE-4.5-based language model) that parses a whole page in one model rather than orchestrating separate detectors and recognizers[^4]. The two approaches now coexist — the modular pipeline gives fine-grained coordinates (cell/character boxes); the VLM gives cleaner end-to-end Markdown at the cost of that granularity.

Everything trains and serves natively in Paddle's model format. ONNX export exists and is well-trodden, but it is a secondary path — the primary, best-supported runtime is Paddle Inference.

## Production Notes

**Installation is the first hurdle.** The framework and the OCR library are separate pip packages, and the GPU wheel must match your CUDA/cuDNN. Mismatches surface as import errors or silent CPU fallback rather than clear messages. Pin both `paddlepaddle` and `paddleocr` versions; do not let them float in CI.

**First-run model downloads.** Models are fetched lazily from remote mirrors on first use. In locked-down or air-gapped environments this fails; pre-download model files and point the OCR object at a local model directory. Mirror selection (Baidu vs HuggingFace vs ModelScope) matters behind restrictive firewalls.

**Version churn is real.** The 2.x → 3.x transition changed the Python API (`ocr.ocr(...)` gave way to `.predict(...)`), renamed parameters, and moved pipeline configuration around. Code and tutorials written for 2.x do not run unmodified on 3.x. Treat a major-version bump as a migration, not an upgrade, and read the changelog before bumping.

**CPU is viable, GPU is faster.** The tiny/small PP-OCR tiers are genuinely usable on CPU for low-throughput workloads — a real advantage over VLM-only stacks. For batch throughput, the project supports OpenVINO, ONNX Runtime, and TensorRT backends plus multi-GPU/multi-process inference; expect to spend time tuning these rather than getting them for free.

**Accuracy is uneven across languages.** Chinese and English are excellent; the long tail of "100+ languages" ranges from strong (major Latin/Cyrillic scripts) to rough (rarer scripts). Benchmark on your own data before committing to a non-CJK, non-Latin language.

**Serving.** There is a service-oriented deployment path (Docker images, HTTP endpoints, SDKs) so non-Python clients (C++, C#, Java) can call OCR over a network boundary. It works but is more assembly-required than a hosted cloud OCR API.

## When to Use / When Not

**Use when:**
- You need strong Chinese/CJK or broad multilingual OCR in a self-hosted, offline, or on-prem setting.
- You want document parsing (tables, layout, KIE) that emits Markdown/JSON for RAG or LLM ingestion.
- You need small models for edge, mobile, or CPU-only deployment.
- Apache-2.0 licensing and no per-call cloud fees matter.

**Avoid when:**
- Your stack is committed to PyTorch and the PaddlePaddle dependency is a non-starter.
- A managed cloud OCR (AWS Textract, Google Document AI, Azure) is acceptable and you'd rather not operate models.
- You only need simple Latin-script OCR — Tesseract or EasyOCR install more easily.
- You can't tolerate API-breaking changes across major versions on your maintenance timeline.

## Alternatives

- tesseract-ocr/tesseract — the mature C++ OCR engine; use it when you want a dependency-light CPU OCR for mostly-Latin text and don't need a deep-learning framework.
- JaidedAI/EasyOCR — PyTorch-based, easier to install, 80+ languages; use it when you want ready OCR without adopting PaddlePaddle.
- mindee/doctr — PyTorch/TensorFlow OCR with document understanding; use it when you're already in a PyTorch/TF shop.
- opendatalab/MinerU — PDF-to-Markdown parsing aimed at LLM/RAG; use it when document-to-Markdown is the entire job.
- VikParuchuri/marker — PDF/document to Markdown with a Python-native stack; use it when you want fast PDF conversion without Paddle.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2020-05 | Repo created; PP-OCR lightweight detection + recognition pipeline[^2]. |
| 2.x | 2020–2023 | Long-lived stable line: PP-OCRv2/v3/v4, PP-Structure (layout, table, KIE), multilingual expansion. |
| 3.x | 2025 | Major reorganization onto PaddleX pipelines; PP-OCRv5. |
| 3.3.0 | 2025-10-16 | PaddleOCR-VL — end-to-end document VLM on a compact ERNIE-4.5 base[^4]. |
| 3.5.0 | 2026-04-21 | Switchable backends (Paddle graph / Transformers / ONNX); PaddleOCR.js browser SDK. |
| 3.6.0 | 2026-05-28 | PaddleOCR-VL-1.6 document parsing model. |
| 3.7.0 | 2026-06-11 | PP-OCRv6 — 50-language unified recognition model in three size tiers. |

## References

[^1]: PaddlePaddle/PaddleOCR repository. https://github.com/PaddlePaddle/PaddleOCR
[^2]: Yuning Du et al., "PP-OCR: A Practical Ultra Lightweight OCR System" — 2020. https://arxiv.org/abs/2009.09941
[^3]: PaddleOCR installation documentation. https://paddlepaddle.github.io/PaddleOCR/latest/en/version3.x/installation.html
[^4]: PaddleOCR-VL documentation. https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/PaddleOCR-VL.html

## Tags

python, ocr, document-parsing, table-recognition, kie, multilingual, chinese-ocr, paddlepaddle, deep-learning, rag, pdf-to-markdown, edge-deployment
