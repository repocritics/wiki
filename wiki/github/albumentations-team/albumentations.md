# albumentations-team/albumentations

> Fast image augmentation library for computer vision — now frozen at its final MIT release, with development moved to a dual-licensed successor.

[GitHub repo](https://github.com/albumentations-team/albumentations) ·
[Official website](https://albumentations.ai) ·
[License: MIT](https://github.com/albumentations-team/albumentations/blob/main/LICENSE)

## Overview

Albumentations is a Python image-augmentation library built for deep-learning training pipelines. It applies randomized transforms — geometric warps, color/noise perturbations, dropout — to images while keeping *associated* targets (segmentation masks, bounding boxes, keypoints, and 3D volumes) consistent under the same spatial transform. That coupled-target guarantee, plus a large transform catalog (70+ operations) and an OpenCV/NumPy backend tuned for speed, is why it became a default choice in Kaggle competition code and research repos from roughly 2019 onward[^1]. It grew out of the competition community: its authors are Kaggle Grandmasters/Masters, and the design biases toward what wins benchmarks — throughput and breadth of transforms — over framework purity.

The defining fact about this repository as of 2026 is that **it is archived and no longer maintained**. The maintainers explicitly ended active development; the last release landed in mid-2025, and the README states there will be no further bug fixes, features, or Python/PyTorch-compatibility updates[^2]. All development moved to a separate project, **AlbumentationsX**, which is a drop-in replacement (`import albumentations as A` still works) but ships under **dual AGPL-3.0 / commercial licensing** rather than MIT[^3].

The tension is now a licensing decision, not a technical one. The code in *this* repo is permissively MIT-licensed and free forever, but slowly rotting: it will eventually break against newer NumPy/PyTorch/Python releases and no one will fix it. The maintained successor is fast-moving but AGPL — a copyleft license incompatible with MIT/Apache/BSD projects, meaning permissive-licensed and proprietary users need a paid commercial license to stay current. Evaluating Albumentations in 2026 is largely about which side of that line you land on.

## Getting Started

```bash
# The frozen, permissive (MIT) library documented here:
pip install -U albumentations

# The maintained successor (AGPL-3.0 / commercial — different license):
pip install albumentationsx
```

```python
import albumentations as A
import cv2

# A pipeline; each transform carries a probability p.
transform = A.Compose([
    A.RandomCrop(width=256, height=256),
    A.HorizontalFlip(p=0.5),
    A.RandomBrightnessContrast(p=0.2),
])

image = cv2.imread("image.jpg")
image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)  # library expects RGB, cv2 loads BGR

transformed = transform(image=image)["image"]
```

For detection/segmentation, pass extra targets and declare their format so they are transformed in lockstep:

```python
transform = A.Compose(
    [A.HorizontalFlip(p=0.5), A.RandomResizedCrop(size=(512, 512))],
    bbox_params=A.BboxParams(format="pascal_voc", label_fields=["class_labels"]),
)
out = transform(image=image, bboxes=bboxes, class_labels=labels)
```

## Architecture / How It Works

The core abstraction is a `Transform` with three probability-gated stages: sample parameters, apply to the image, apply the *same* sampled parameters to each additional target. `A.Compose` chains transforms and owns the target-format bookkeeping (`bbox_params`, `keypoint_params`). Meta-transforms like `A.OneOf` and `A.SomeOf` compose sub-pipelines with their own selection probabilities. This design is what makes mask/bbox/keypoint consistency automatic rather than the caller's problem.

Transforms split into two families the docs treat as first-class: **pixel-level** (change only the image — blur, noise, color, compression) and **spatial-level** (change image *and* all geometric targets — affine, crop, flip, distortion). Only spatial transforms touch masks/boxes/keypoints; pixel transforms leave them untouched. Volumetric (3D) data is handled by applying 2D transforms slice-by-slice along the depth axis.

The compute backend is OpenCV and NumPy, not a tensor framework. Augmentation runs on CPU, on NumPy `uint8`/`float32` arrays, typically inside a PyTorch `Dataset.__getitem__` on DataLoader worker processes. Albumentations is framework-agnostic by design — it emits arrays and leaves tensor conversion to a thin adapter (`ToTensorV2`). Its historical speed advantage came from routing operations through OpenCV's SIMD-optimized C++ primitives rather than pure-Python or PIL paths, and the project maintained public benchmarks asserting it as the fastest option for common transforms[^1].

## Production Notes

- **This repo is a dependency-freeze target, not a living library.** Pinning `albumentations` is safe for reproducibility but accrues risk: a future NumPy 2.x or PyTorch release can break it with no upstream fix coming[^2]. Budget for either staying on old pins or migrating.
- **The migration path is a relicense, not a rewrite.** `pip install albumentationsx` keeps the `import albumentations` name and API, so code changes are near-zero — but AlbumentationsX is **AGPL-3.0**. For permissive-licensed OSS (MIT/Apache/BSD) or proprietary/SaaS use, AGPL's network-copyleft is usually a non-starter and a commercial license is required[^3]. Treat the "drop-in" claim as technically true and legally significant.
- **CPU augmentation is a throughput bottleneck.** Because transforms run on CPU DataLoader workers, heavy pipelines can starve the GPU on fast models. Common mitigations: raise `num_workers`, prune expensive transforms (`ElasticTransform`, `GridDistortion`, `Superpixels` are costly), or move augmentation onto the GPU with a different library (Kornia, NVIDIA DALI).
- **Color-space and dtype footguns.** OpenCV loads BGR; the library expects RGB — forgetting `cvtColor` silently trains on channel-swapped data. Some transforms assume `uint8` [0,255], others `float32` [0,1]; `Normalize`/`ToFloat`/`FromFloat` placement in the pipeline matters.
- **API churn predates the archive.** The 1.0 line (2021) and subsequent minors renamed/removed parameters and transforms; older tutorials and Stack Overflow answers frequently reference signatures that no longer exist. Now that the repo is frozen, this stops getting worse but existing code pinned to old versions won't get compatibility shims.

## When to Use / When Not

**Use when:**
- You need broad CPU-side augmentation with automatic mask/bbox/keypoint consistency and are fine pinning a stable, unchanging version.
- Your project is itself GPL/AGPL-compatible (then AlbumentationsX is free) *or* you simply need the existing MIT code to keep working as-is.
- You're reproducing a paper or competition solution that already depends on it.

**Avoid when:**
- You need an actively maintained dependency with ongoing Python/PyTorch compatibility — use AlbumentationsX (with the license caveat) or another live library.
- You're MIT/Apache/proprietary and unwilling to pay for a commercial license to get updates — the AGPL successor won't fit and this repo will bit-rot.
- Augmentation is your GPU-starvation bottleneck — a GPU-native library will serve better.

## Alternatives

- albumentations-team/AlbumentationsX — the official maintained successor; same API, dual AGPL-3.0/commercial license.
- kornia/kornia — differentiable, GPU-native augmentation as PyTorch ops; use when you want augmentation on-device and in the autograd graph.
- NVIDIA/DALI — GPU/CPU data-loading + augmentation pipeline; use for high-throughput training bottlenecked on the input pipeline.
- pytorch/vision (`torchvision.transforms.v2`) — first-party PyTorch transforms with bbox/mask support; use when you want to minimize dependencies and stay in-framework.
- aleju/imgaug — the older, broad augmentation library Albumentations partly displaced; itself largely unmaintained now.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2018-06 | Public release; paper published in *Information* (MDPI)[^4]. |
| 1.0.0 | 2021-07 | API stabilization; parameter/transform renames and removals. |
| 1.4.x | 2024 | 3D/volumetric targets, expanded transform catalog. |
| 2.0.x | 2025 | Final maintained MIT line under this repo. |
| (archived) | 2025-06 | Last commit; repo frozen, development moves to AlbumentationsX[^2]. |

## References

[^1]: Buslaev et al., "Albumentations: Fast and Flexible Image Augmentations," *Information* 11(2):125, 2020. https://www.mdpi.com/2078-2489/11/2/125 — benchmarks and design goals. Project benchmarks: https://albumentations.ai/docs/benchmarks/image-benchmarks/
[^2]: Repository README, "Important Notice: Albumentations is No Longer Maintained" — last update June 2025, no further fixes/features. https://github.com/albumentations-team/albumentations
[^3]: AlbumentationsX — successor project, dual AGPL-3.0 / commercial licensing; drop-in `pip install albumentationsx`. https://github.com/albumentations-team/AlbumentationsX and pricing at https://albumentations.ai/pricing
[^4]: Same as [^1]; MDPI *Information* publication, 2020.

## Tags

python, image-augmentation, computer-vision, deep-learning, data-augmentation, object-detection, image-segmentation, opencv, machine-learning, unmaintained
