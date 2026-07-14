# facebookresearch/segment-anything

> Meta AI's promptable image segmentation model (SAM) with inference code, pretrained checkpoints, and example notebooks — a research reference release, now superseded by SAM 2.

[GitHub repo](https://github.com/facebookresearch/segment-anything) ·
[Official website](https://segment-anything.com) ·
[License: Apache-2.0](https://github.com/facebookresearch/segment-anything/blob/main/LICENSE)

## Overview

Segment Anything is the public code release accompanying Meta AI (FAIR)'s 2023 paper "Segment Anything"[^1]. It ships inference code for the Segment Anything Model (SAM), download links for three pretrained checkpoints, and Jupyter notebooks demonstrating usage. SAM takes an image plus a prompt — points, a bounding box, or a coarse mask — and returns high-quality object masks. It was trained on SA-1B, a dataset of 11 million images and 1.1 billion masks, and its selling point is strong zero-shot generalization: it segments objects it was never explicitly trained on, without task-specific fine-tuning.

The defining characteristic of this repository is that it is a research artifact, not a maintained product. There is no training code, no fine-tuning harness, and no PyPI package — you install from a git URL and load a checkpoint. It is inference-only reference code that demonstrates the model in the paper. The repository is not formally archived, but its last substantive commit was in September 2024, when a pointer to SAM 2 was added; new work should target SAM 2 (`facebookresearch/sam2`), which extends the same idea to video and is faster[^2].

The intended audience is computer-vision researchers and engineers building segmentation pipelines who want a strong zero-shot mask proposer as a component. It is not an end-user application, though the hosted demo at segment-anything.com and numerous downstream projects (annotation tools, text-prompted variants) make it accessible in practice.

## Getting Started

```bash
# No PyPI release — install from the git repository
pip install git+https://github.com/facebookresearch/segment-anything.git

# Optional deps for mask post-processing, COCO export, notebooks, ONNX
pip install opencv-python pycocotools matplotlib onnxruntime onnx
```

Download a checkpoint first (e.g. `vit_h`, ~2.4 GB), then:

```python
import cv2
from segment_anything import SamPredictor, sam_model_registry

sam = sam_model_registry["vit_h"](checkpoint="sam_vit_h_4b8939.pth")
sam.to("cuda")                       # CPU works but is very slow
predictor = SamPredictor(sam)

image = cv2.cvtColor(cv2.imread("image.jpg"), cv2.COLOR_BGR2RGB)
predictor.set_image(image)           # runs the heavy image encoder once

# Prompt with a single foreground point (x, y); label 1 = foreground
masks, scores, logits = predictor.predict(
    point_coords=[[500, 375]],
    point_labels=[1],
    multimask_output=True,           # returns 3 masks for ambiguous prompts
)
```

To segment every object in an image without prompts, use `SamAutomaticMaskGenerator`, which tiles a grid of point prompts across the image and deduplicates the results.

## Architecture / How It Works

SAM is three components with deliberately asymmetric cost:

1. **Image encoder** — an MAE-pretrained Vision Transformer (ViT) backbone. This is the expensive part. It runs once per image via `predictor.set_image()`, producing an image embedding. ViT-H (the `default`) is the largest; ViT-L and ViT-B trade accuracy for speed and memory.
2. **Prompt encoder** — lightweight; encodes points, boxes, and coarse masks into embeddings. (The paper describes a text prompt path, but public checkpoints do not ship a usable text encoder — text prompting in the wild is done by bolting on an external detector.)
3. **Mask decoder** — a small, fast transformer that fuses the image embedding with prompt embeddings and emits masks. It is light enough to run at interactive rates, and can be exported to ONNX and run in-browser, which is how the web demo achieves real-time interaction.

The asymmetry is the key design insight: encode the image once (slow, on a GPU), then answer many prompts cheaply (fast, even client-side). For ambiguous prompts — a single point on a person's shirt could mean the shirt, the torso, or the whole person — SAM returns three masks with self-predicted IoU scores rather than guessing one. Each mask also carries a `stability_score`. Callers pick or threshold among them.

`SamAutomaticMaskGenerator` builds on this by sampling a regular point grid, running the decoder per point, and filtering by predicted IoU and stability, then applying non-maximal suppression. It is convenient but slow and produces many overlapping masks that need downstream reconciliation.

## Production Notes

- **The image encoder is the bottleneck, not the decoder.** ViT-H inference is heavy; expect a GPU requirement for anything interactive. On CPU, `set_image()` can take seconds to tens of seconds per image. Batch or cache image embeddings if you re-prompt the same image.
- **No official pip/conda package.** Installation is from a git URL, which pins you to a commit rather than a semver release. Vendoring the repo or pinning a git SHA is advisable for reproducible builds.
- **Checkpoints are large and externally hosted.** ViT-H is ~2.4 GB, served from `dl.fbaipublicfiles.com` — not a package registry, not a CDN with an SLA. Mirror the weights internally for production.
- **License split matters.** The code and model weights are Apache-2.0, but the SA-1B training dataset is under a separate research-only license. If you retrain or redistribute derived from SA-1B, the permissive Apache terms on the code do not cover the data.
- **Inference only.** There is no fine-tuning or training code here. Adapting SAM to a domain (medical imaging, satellite) requires third-party projects (e.g. HQ-SAM, MedSAM) or your own harness.
- **Automatic mask generation is not free.** The grid-prompt approach multiplies decoder calls by the number of grid points and produces overlapping, unlabeled masks. It is a proposer, not a semantic segmenter — there are no class labels.
- **Prefer SAM 2 for new work.** SAM 2 is faster on images, handles video, and is where Meta's continued effort went. This repo is effectively frozen; open issues (~600) are largely unaddressed[^2].

## When to Use / When Not

**Use when:**
- You need zero-shot object masks from point or box prompts as a pipeline component.
- You are reproducing or extending the original SAM paper specifically.
- You want an interactive mask tool whose decoder can run client-side via ONNX.
- You are prompting with known geometry (e.g. box outputs from a detector) and want tight masks.

**Avoid when:**
- You need class labels or semantic segmentation — SAM produces masks, not categories.
- You need video, or the best speed/accuracy available — use SAM 2 instead.
- You want a maintained dependency with releases and active issue triage.
- You are CPU-bound with real-time requirements — the ViT encoder will not keep up.
- You need out-of-the-box text prompting — that requires an external detector.

## Alternatives

- `facebookresearch/sam2` — the direct successor; use instead for images and video, faster, actively developed.
- `IDEA-Research/Grounded-Segment-Anything` — use when you need text-prompted segmentation (GroundingDINO detects, SAM segments).
- `ChaoningZhang/MobileSAM` — use when you need SAM-style masks on edge/CPU; a distilled lightweight image encoder.
- `CASIA-IVA-Lab/FastSAM` — use when speed matters more than mask fidelity; a YOLOv8-seg-based approximation.
- `SysCV/sam-hq` — use when you need higher-quality mask boundaries than stock SAM produces.

## History

| Version | Date | Notes |
|---------|------|-------|
| Paper + release | 2023-04 | "Segment Anything" (arXiv:2304.02643); repo public, SA-1B, three ViT checkpoints[^1]. |
| ONNX / web demo | 2023 | Lightweight decoder exportable to ONNX; in-browser demo. |
| SAM 2 pointer | 2024-09 | README updated to direct users to SAM 2; last substantive commit[^2]. |

The repository does not use tagged semver releases; it is a research code drop with incremental commits rather than versioned artifacts.

## References

[^1]: Kirillov et al., "Segment Anything" — Meta AI Research (FAIR), arXiv:2304.02643, 2023. https://arxiv.org/abs/2304.02643
[^2]: Segment Anything Model 2 (SAM 2). https://github.com/facebookresearch/sam2 · paper: https://arxiv.org/abs/2408.00714

## Tags

computer-vision, image-segmentation, foundation-model, zero-shot, pytorch, vision-transformer, onnx, meta-ai, research-code, inference-only
