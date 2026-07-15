# facebookresearch/detectron2

> Facebook AI Research's PyTorch platform for object detection, instance/panoptic segmentation, and keypoint estimation — a research codebase that became an industry default.

[GitHub repo](https://github.com/facebookresearch/detectron2) ·
[Documentation](https://detectron2.readthedocs.io/en/latest/) ·
[License: Apache-2.0](https://github.com/facebookresearch/detectron2/blob/main/LICENSE)

## Overview

Detectron2 is FAIR's second-generation detection library, released in 2019 as the successor to the Caffe2-based Detectron and to maskrcnn-benchmark[^1]. It reimplements the same family of detectors — Faster R-CNN, Mask R-CNN, RetinaNet, and later additions like Cascade R-CNN, PointRend, DensePose, panoptic FPN, DeepLab, and ViTDet — on pure PyTorch, with a registry-and-config design meant to let researchers swap components without forking the whole pipeline[^2].

It is used two ways that pull in different directions. As a *library*, it is imported by downstream research projects (many live in `projects/` in-tree) that reuse its backbones, ROI heads, and training loop. As an *application*, it ships a Model Zoo of pretrained COCO/LVIS baselines and a `DefaultTrainer`/`DefaultPredictor` that let you fine-tune or run inference with a config file and a few lines[^3]. The defining tension is maturity versus momentum: the API is stable and battle-tested, but the project has been in de-facto maintenance mode since the v0.6 release in late 2021 — commits still land, but there have been no new tagged releases and the open-issue count stays high. Newer detection research has largely moved to transformer-based codebases and to mmdetection.

## Getting Started

There are no current prebuilt wheels; install from source against your existing PyTorch/CUDA:

```bash
python -m pip install 'git+https://github.com/facebookresearch/detectron2.git'
# requires a matching torch + torchvision already installed,
# and a CUDA toolkit that matches your torch build for GPU ops
```

```python
from detectron2 import model_zoo
from detectron2.config import get_cfg
from detectron2.engine import DefaultPredictor
import cv2

cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml")
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5

predictor = DefaultPredictor(cfg)          # single-image inference
outputs = predictor(cv2.imread("input.jpg"))
print(outputs["instances"].pred_classes, outputs["instances"].pred_boxes)
```

## Architecture / How It Works

The core idea is a **registry** system. Backbones, proposal generators, ROI heads, and whole meta-architectures register themselves by name (`BACKBONE_REGISTRY`, `META_ARCH_REGISTRY`, `ROI_HEADS_REGISTRY`, …), and `build_model(cfg)` assembles a network by looking up the string names in the config[^2]. A `GeneralizedRCNN` is literally `backbone → proposal_generator (RPN) → roi_heads`, each independently replaceable. This is what makes it a *platform*: a research project can register one new head and inherit everything else.

Configuration has two generations. The original is **yacs `CfgNode`** — a nested YAML tree (`cfg.MODEL.ROI_HEADS.NUM_CLASSES = 80`) merged from a base file. Every model in the Model Zoo is a YAML config. Later versions added **LazyConfig**, a Python-based config where the config is an actual object graph you edit in `.py` files; the "new baselines" and ViTDet configs use it[^4]. The two systems coexist and are not interchangeable — a footgun when copying snippets across eras of the codebase.

Data flows through `DatasetCatalog` and `MetadataCatalog`: you register a dataset name mapped to a function returning a list of per-image dicts, plus metadata (class names, colors). `DatasetMapper` turns those dicts into tensors, and `build_detection_train_loader` produces batches. Training runs through `DefaultTrainer`, an opinionated loop built on a hook system (`HookBase`) for checkpointing, evaluation, and logging; heavy customization usually means subclassing it or dropping to the lower-level `SimpleTrainer`.

Performance-critical operations — ROIAlign, deformable convolution, rotated NMS, `nms` — are custom C++/CUDA kernels compiled at install time (historically shared with fvcore). This is the source of most install pain: the compiled ops are pinned to the exact torch ABI and CUDA version present when the package was built.

## Production Notes

- **Installation is the number-one issue.** The compiled CUDA ops must be built against your specific torch + CUDA combination. Old prebuilt wheels existed only for a fixed matrix of torch 1.x / CUDA versions and are no longer updated, so on modern PyTorch you build from source and need a CUDA toolkit matching your torch build. Version-mismatch errors (`undefined symbol`, ABI mismatch) are extremely common. Pin torch first, then install detectron2 against it.
- **Maintenance status.** The last tagged release is v0.6 (November 2021). The repo still receives occasional commits, but treat it as feature-frozen: do not expect support for the newest PyTorch, CUDA, or Python versions to land promptly, and expect the ~500+ open issues to include unaddressed environment breakages.
- **`DefaultPredictor` is single-image and slow for serving.** It re-does preprocessing per call and runs one image at a time. For throughput you build your own batched inference on top of the raw model plus `DatasetMapper`, or export the model.
- **Export has caveats.** TorchScript export via `TracingAdapter`/`scripting` works for supported meta-architectures but tracing bakes in input assumptions; dynamic shapes and some postprocessing don't trace cleanly. The older Caffe2 export path is deprecated. There is no first-class ONNX story for the full pipeline.
- **Windows is community-supported only.** Official CI targets Linux; expect to fight the build on Windows.
- **Model Zoo weights are pickles.** Checkpoints are `.pkl`/`.pth` in detectron2's own format; loading them elsewhere requires the matching architecture definition, not just the weights.

## When to Use / When Not

**Use when:**
- You need proven Mask R-CNN / Faster R-CNN / RetinaNet baselines with a curated COCO/LVIS Model Zoo.
- You are building a research project that benefits from swapping one component (a head, a backbone) while reusing a stable training loop.
- You value API stability over having the latest architectures.
- You already run a pinned PyTorch/CUDA environment and can build from source.

**Avoid when:**
- You need active upstream maintenance and support for the newest PyTorch/CUDA — it is effectively frozen at v0.6.
- You want real-time or edge inference — a YOLO-family library is faster to deploy and lighter.
- You want the broadest, most current model zoo — mmdetection tracks more recent detectors.
- You need clean ONNX/TensorRT export — the export paths are partial and dated.

## Alternatives

- open-mmlab/mmdetection — use instead when you want a larger, more actively maintained zoo of modern detectors, at the cost of an even heavier config system.
- ultralytics/ultralytics — use for real-time/edge object detection and segmentation with a simpler API and export story.
- facebookresearch/Mask2Former — use for state-of-the-art unified panoptic/instance/semantic segmentation (itself built on detectron2).
- pytorch/vision — use when you want a few detection models with minimal dependencies inside the core torchvision package.
- PaddlePaddle/PaddleDetection — use when you are in the PaddlePaddle ecosystem or want strong deployment tooling.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-10 | Public release; PyTorch rewrite of Detectron/maskrcnn-benchmark[^1]. |
| 0.1 | 2020-02 | First tagged release. |
| 0.2 | 2020-07 | PointRend, DensePose improvements, TorchScript work. |
| 0.3 | 2020-11 | DeepLab, panoptic/semantic segmentation additions. |
| 0.4 | 2021-03 | LazyConfig introduced, new-baseline configs[^4]. |
| 0.5 | 2021-07 | ViTDet/MViTv2-era additions, more new baselines. |
| 0.6 | 2021-11 | Latest tagged release; project since in maintenance mode. |

## References

[^1]: Detectron2 README and repository — FAIR. https://github.com/facebookresearch/detectron2
[^2]: Yuxin Wu, Alexander Kirillov, Francisco Massa, Wan-Yen Lo, Ross Girshick, "Detectron2" (2019). https://github.com/facebookresearch/detectron2
[^3]: Getting Started with Detectron2. https://detectron2.readthedocs.io/en/latest/tutorials/getting_started.html
[^4]: LazyConfig system documentation. https://detectron2.readthedocs.io/en/latest/tutorials/lazyconfigs.html

## Tags

python, pytorch, computer-vision, object-detection, instance-segmentation, panoptic-segmentation, mask-rcnn, deep-learning, machine-learning, model-zoo
