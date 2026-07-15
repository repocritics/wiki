# kornia/kornia

> A differentiable computer vision library for PyTorch — classical CV operators rewritten as autograd-friendly tensor ops.

[GitHub repo](https://github.com/kornia/kornia) ·
[Official website](https://kornia.readthedocs.io) ·
[License: Apache-2.0](https://github.com/kornia/kornia/blob/main/LICENSE)

## Overview

Kornia is a computer vision library built entirely on top of PyTorch tensors, so that classical vision operations — color conversions, geometric warps, filters, feature detectors, epipolar geometry — are all differentiable and run on the GPU inside an autograd graph[^1]. It grew out of the earlier `torchgeometry` (PyTorch Geometry) project and was formalized in a WACV 2020 paper by Riba, Mishkin, Ponsa, Rublee, and Bradski[^2]. The last name is not incidental: Gary Bradski originated OpenCV, and Kornia is often described as "OpenCV that backpropagates."

The defining tradeoff is exactly that positioning. Because every operator is a composition of PyTorch ops, gradients flow through a homography estimation or a Sobel filter the same way they flow through a convolution, which makes Kornia useful for end-to-end trainable vision pipelines, differentiable data augmentation, and self-supervised geometry. The cost is that Kornia inherits PyTorch's performance envelope and dtype constraints rather than the hand-tuned SIMD/threaded C++ of OpenCV. For a one-shot CPU resize, OpenCV is faster and lighter; Kornia earns its keep when the operation sits inside a training loop or needs to be batched on a GPU.

As of 2026 the project has explicitly announced a strategic shift "towards end-to-end vision models," prioritizing integration of Vision Language Models (VLM) and Vision Language Agents (VLA) alongside the classical operator core[^3]. This is a meaningful scope expansion for a library whose historical value proposition was low-level differentiable primitives, and it is worth watching whether the operator library continues to get the same maintenance attention.

## Getting Started

```bash
pip install kornia
```

```python
import torch
import kornia as K

# A batch of images as a float tensor in [0,1], shape (B, C, H, W)
img = torch.rand(4, 3, 256, 256)

# Everything below is differentiable and runs on whatever device `img` is on.
gray = K.color.rgb_to_grayscale(img)          # (4, 1, 256, 256)
blurred = K.filters.gaussian_blur2d(img, (5, 5), (1.5, 1.5))
edges = K.filters.sobel(gray)

# Geometric warp with a batch of 3x3 homographies
H = torch.eye(3).repeat(4, 1, 1)
warped = K.geometry.transform.warp_perspective(img, H, dsize=(256, 256))
```

Differentiable augmentation composes like `nn.Module` layers and is meant to live on the GPU inside the training step, not in the dataloader:

```python
from kornia.augmentation import AugmentationSequential, RandomAffine, RandomBrightness

aug = AugmentationSequential(
    RandomAffine(degrees=(-45.0, 45.0), p=1.0),
    RandomBrightness(brightness=(0.0, 1.0), p=1.0),
)
out = aug(img)          # gradients flow; transform params are sampled per-batch
```

## Architecture / How It Works

Kornia is organized as a set of submodules that each cover a slice of classical CV, all expressed as `Tensor -> Tensor` functions plus `nn.Module` wrappers:

- `kornia.color` — colorspace conversions (RGB, HSV, LAB, YUV, grayscale).
- `kornia.filters` — Gaussian, Sobel, Laplacian, median, box, bilateral, guided, unsharp; most implemented as separable convolutions.
- `kornia.geometry` — the largest and most distinctive area: `transform` (affine/perspective warps, resize, rotate), `camera` (pinhole, calibration), `epipolar` (fundamental/essential matrices, triangulation), `homography`, `liegroup` (SO3/SE3), and solvers.
- `kornia.feature` — local features and matching: Harris/GFTT/KeyNet detectors, SIFT/HardNet/SOSNet descriptors, and learned matchers LoFTR, LightGlue, DISK, DeDoDe.
- `kornia.augmentation` — the container-based augmentation engine (`AugmentationSequential`, `PatchSequential`, `VideoSequential`) that tracks and can invert the sampled transforms.
- `kornia.enhance`, `kornia.morphology`, `kornia.losses`, `kornia.metrics`, `kornia.contrib`, `kornia.models`.

The core design invariant is the batched NCHW float tensor. Operators expect `(B, C, H, W)` in floating point, typically normalized to `[0, 1]`, and preserve differentiability end to end. Where an operation needs numerically sensitive linear algebra (SVD, matrix inverse, PnP), Kornia routes through internal cast helpers (`_torch_svd_cast`, `_torch_solve_cast`) so that reduced-precision inputs get promoted to fp32/fp64 for the unstable step.

Image I/O is deliberately not part of the tensor library. A companion Rust crate, `kornia-rs`, provides fast image decode/encode and resize (`read_image_any`, `resize`) returning arrays that feed into the tensor pipeline. There is also an `ONNXSequential` path that chains Kornia operators and models into exportable ONNX graphs (including `hf://` references to Hugging Face-hosted operators), and an experimental multi-framework bridge (`kornia.to_tensorflow()`) built on the Ivy transpiler to expose operators to TensorFlow/JAX/NumPy backends.

## Production Notes

- **Reduced precision is partial, and documented as such.** The maintainers ship a per-module fp16/bf16 support matrix rather than claiming blanket support. Pure conv/pool modules (`morphology`) are fully typed; `geometry.calibration` (the PnP solver) explicitly accepts only fp32/fp64; FFT-based and linalg-heavy paths (some `color`, `filters`, `homography`, `epipolar`) fail or lose accuracy in half precision. If you train in AMP/bf16, test the specific operators you depend on — a published test run showed the CPU fp16 suite at ~90% pass vs ~99.9% for fp32[^4].
- **It is not a drop-in OpenCV replacement.** APIs, coordinate conventions, and channel ordering differ; Kornia is NCHW float tensors, OpenCV is HWC uint8 numpy. Porting a pipeline means rewriting, not aliasing. Reach for Kornia when you need gradients or GPU batching, not to avoid an OpenCV dependency.
- **Augmentations belong on the GPU, not in the DataLoader.** The augmentation module's payoff is running batched transforms on-device inside the training step. Placing it in per-sample CPU dataloader workers forfeits most of the benefit and can be slower than `torchvision.transforms`.
- **PyTorch coupling drives the support window.** Kornia tracks a minimum PyTorch (2.0.0+ on recent releases) and follows PyTorch's own device/dtype behavior. Upgrades are usually smooth, but the library's floor moves with PyTorch's, and some feature models pull heavy optional dependencies (diffusers, ONNX runtime, transformers) that are not installed by the base `pip install kornia`.
- **The 0.x version number is real.** Despite years of production use and a large operator surface, Kornia has never declared 1.0. Minor releases (0.6 → 0.7 → 0.8) have carried behavioral and signature changes; pin the version and read release notes before upgrading.
- **Volunteer-maintained.** Development is community-driven with an OpenCollective for sponsorship. Response time on niche operators varies, and the recent VLM/VLA pivot may shift contributor attention away from the classical core.

## When to Use / When Not

**Use when:**
- You need vision operators *inside* an autograd graph — differentiable augmentation, self-supervised geometry, learnable warps, photometric losses.
- You want batched, GPU-resident classical CV that shares device and dtype with your model.
- You need learned local-feature matching (LoFTR, LightGlue, DISK) with a consistent PyTorch API.
- Your pipeline is already PyTorch and you want to keep everything as tensors end to end.

**Avoid when:**
- You just need fast one-off image processing with no gradients — OpenCV or Pillow-SIMD is lighter and faster.
- You are CPU-bound on uint8 images; Kornia's float-tensor model adds overhead.
- You need the full breadth of OpenCV's classical algorithms (many contrib modules have no Kornia equivalent).
- You require strict half-precision throughout — several linalg/FFT paths are fp32-only.

## Alternatives

- opencv/opencv — the classical reference; faster for non-differentiable CPU work, vastly larger algorithm catalog, but no autograd and no native batching.
- pytorch/vision (torchvision) — overlapping transforms/ops and pretrained models within the PyTorch ecosystem; use it when you need standard augmentation and model zoo rather than differentiable geometry.
- albumentations-team/albumentations — faster, richer CPU augmentation for the dataloader; use it when augmentation need not be differentiable or on-GPU.
- scikit-image/scikit-image — broad NumPy-based image processing for scientific work; use when you are not in PyTorch and do not need gradients.
- colmap/colmap — dedicated Structure-from-Motion / MVS pipeline; use for full 3D reconstruction rather than differentiable geometry primitives.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019 | Formalized from `torchgeometry`; WACV 2020 paper[^2]. |
| 0.5.x | 2021 | Expanded feature/geometry modules. |
| 0.6.0 | 2021-10-22 | Major minor line; broad operator growth. |
| 0.6.8 | 2022-10-13 | Continued feature-matching additions (LoFTR era). |
| 0.7.0 | 2023-08-02 | New minor line. |
| 0.7.4 | 2024-11-05 | Last of the 0.7 series. |
| 0.8.0 | 2025-01-11 | 0.8 line; PyTorch 2.x baseline. |
| 0.8.3 | 2026-05-19 | Latest release; VLM/VLA end-to-end direction announced[^3]. |

## References

[^1]: Kornia README and documentation. https://kornia.readthedocs.io
[^2]: E. Riba, D. Mishkin, D. Ponsa, E. Rublee, G. Bradski, "Kornia: an Open Source Differentiable Computer Vision Library for PyTorch," WACV 2020. https://arxiv.org/abs/1910.02190
[^3]: Kornia project announcement on shift toward end-to-end VLM/VLA vision models — README, 2026. https://github.com/kornia/kornia
[^4]: Kornia half-precision support matrix and test results (commit 6131e98, 2026-03-21), README / precision guide. https://kornia.readthedocs.io/en/stable/get-started/precision.html

## Tags

python, pytorch, computer-vision, differentiable, image-processing, deep-learning, geometry, feature-matching, gpu, data-augmentation
