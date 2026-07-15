# open-mmlab/mmdetection

> A config-driven PyTorch object detection toolbox and benchmark — dozens of published detectors reimplemented against one shared registry.

[GitHub repo](https://github.com/open-mmlab/mmdetection) ·
[Documentation](https://mmdetection.readthedocs.io) ·
[License: Apache-2.0](https://github.com/open-mmlab/mmdetection/blob/main/LICENSE)

## Overview

MMDetection is the object-detection component of OpenMMLab, an academic-origin family of PyTorch vision toolkits from the Shanghai AI Laboratory / MMLab lineage[^1]. It began in 2018 as the released codebase behind the team's 2018 COCO Detection Challenge entry and grew into a reference implementation for the detection literature: Faster R-CNN, Mask R-CNN, RetinaNet, Cascade R-CNN, FCOS, DETR and its many successors (Deformable DETR, DINO, DAB-DETR), YOLOX, RTMDet, and open-vocabulary models like GLIP and MM-Grounding-DINO all live in one repo, sharing components and a common training loop[^2].

The defining idea is **config-as-model**. You do not write a training script; you write (or inherit) a Python-dict config that names registered components — backbone, neck, head, dataset, transforms, optimizer, schedule, hooks — and a `Runner` assembles and executes it. This makes MMDetection excellent for reproducing papers and A/B-testing architectural pieces, and frustrating for anyone who wants a plain `model(x)` call. The abstraction that lets you swap a ResNet for a Swin backbone by editing three lines is the same abstraction that turns a stack trace into a walk through the registry.

The second defining tension is versioning. MMDetection sits on top of two sibling libraries — MMCV (CUDA/C++ vision ops) and, since 3.0, MMEngine (the training runtime)[^3]. The three must be version-matched, and the compiled MMCV ops must match your exact PyTorch + CUDA build. Getting that quadruple aligned is the single most common reason a fresh install fails. As of the latest push (August 2024) the repo has been quiet for roughly a year; it is stable and widely used but no longer under fast development.

## Getting Started

Install via OpenMMLab's own package manager, `mim`, which resolves the MMCV/MMEngine version matrix for you:

```bash
pip install -U openmim
mim install mmengine
mim install "mmcv>=2.0.0"
mim install mmdet
```

Run inference with a pretrained model (weights auto-download from the model zoo):

```python
from mmdet.apis import DetInferencer

# RTMDet-tiny; first call downloads the checkpoint
inferencer = DetInferencer(model='rtmdet_tiny_8xb32-300e_coco')
result = inferencer('demo/demo.jpg', out_dir='outputs/', show=False)

# result['predictions'][0] has 'bboxes', 'labels', 'scores'
print(result['predictions'][0]['labels'][:5])
```

Training is config-driven, not code-driven:

```bash
# fine-tune a config from the repo on your own COCO-format data
python tools/train.py configs/rtmdet/rtmdet_s_8xb32-300e_coco.py
```

## Architecture / How It Works

Everything hinges on the **registry + config** pattern:

1. **Registries** — global name→class maps (`MODELS`, `DATASETS`, `TRANSFORMS`, `HOOKS`, `OPTIMIZERS`, …). Every component decorates itself with `@MODELS.register_module()` and becomes constructible by string name.
2. **Configs** — plain Python files (evaluated to dicts) describing an experiment. `_base_ = [...]` gives multiple inheritance; a leaf config overrides fields by key path. A single `type='ResNet'` string is enough to instantiate the registered class with the sibling kwargs.
3. **Runner** (from MMEngine) — owns the train/val/test loops, hooks, optimizer wrapping, mixed precision, distributed launch, checkpointing, and logging. MMDetection contributes detectors and data pipelines; MMEngine drives them.

The model side follows a fixed decomposition: **backbone** (ResNet, Swin, ConvNeXt) → **neck** (FPN and variants) → **dense head** and/or **roi head** → **loss** + **assigner/sampler**. Two-stage detectors (Faster/Mask/Cascade R-CNN) wire an RPN into a roi head; single-stage (RetinaNet, FCOS, RTMDet) and DETR-family models substitute their own heads. Because these are all registry entries, a config can graft, say, a Swin backbone onto a Mask R-CNN neck+head without touching Python.

Performance-critical operations — NMS, RoIAlign, deformable convolution, focal loss — are not pure PyTorch; they are compiled CUDA/C++ kernels shipped in **MMCV**[^3]. This is why MMCV is a hard, version-pinned dependency rather than a `pip`-any wheel: the ops link against a specific PyTorch ABI and CUDA toolkit.

Data flows through a **transform pipeline** — an ordered list of registered transforms (`LoadImageFromFile`, `Resize`, `RandomFlip`, `PackDetInputs`) that each mutate a results dict. Datasets emit COCO-style annotations; `CocoDataset` and `CocoMetric` are the defaults, and custom datasets are typically shoehorned into COCO JSON rather than given bespoke loaders.

## Production Notes

**Dependency alignment is the main footgun.** mmdet, mmcv, and mmengine each declare compatible-version ranges, and mmcv's compiled ops must match your torch+CUDA. Use `mim install` rather than raw `pip`; when it still breaks, the error is usually a prebuilt-mmcv wheel that doesn't exist for your exact torch/CUDA pair, forcing a source build. Pin all three (plus torch) in any reproducible environment.

**The 2.x → 3.x rewrite is not source-compatible.** MMDetection 3.0 moved the runtime onto MMEngine and changed config schema, module paths, and data-structure conventions (the `DetDataSample` / `InstanceData` containers). Configs, custom modules, and checkpoints written for 2.x do not run on 3.x unchanged; an official migration guide exists but expect real porting work[^4]. Much third-party tutorial content still targets 2.x — check the version before copying a config.

**Debuggability cost.** Because instantiation goes through string lookups and dict merges, a typo in a config surfaces as a construction error deep in the registry, not a Python `NameError` at the call site. Tracing "where did this hyperparameter come from" means walking the `_base_` inheritance chain. IDE go-to-definition is largely useless across the config boundary.

**Deployment is a separate stack.** For TensorRT/ONNX/production serving you are pushed toward **MMDeploy**, a sibling repo, rather than exporting from mmdet directly. Plan for that toolchain and its own version matrix if you need anything beyond Python inference.

**Reproducibility is a genuine strength.** The model zoo ships configs *and* matching checkpoints with reported COCO metrics, so "reproduce the paper's number" is realistic in a way it rarely is elsewhere. RTMDet, the team's own real-time family, reports competitive speed/accuracy trade-offs (e.g. COCO detection and instance segmentation) with configs in-repo[^5].

**Maintenance cadence.** The default branch last saw commits in August 2024. Issues remain open in the thousands. Treat it as a mature, slowing project: excellent for the architectures already implemented, not a place where the newest 2025+ detectors will necessarily appear first.

## When to Use / When Not

**Use when:**
- You need to reproduce or benchmark published detectors on a common footing.
- Your research swaps components (backbones, necks, assigners, losses) and you want that to be a config edit, not a fork.
- You want pretrained COCO weights across a very wide catalog of architectures in one place.
- You already run other OpenMMLab toolkits and know the config/registry idiom.

**Avoid when:**
- You want a two-line `pip install` and a `model.predict(img)` API — Ultralytics or a Hugging Face pipeline is far less ceremony.
- You need a small, auditable dependency surface (the MMCV/MMEngine/CUDA coupling is heavy).
- You are shipping to edge/production and don't want a second (MMDeploy) toolchain.
- You need the very latest detectors and active maintenance in 2025–2026.

## Alternatives

- facebookresearch/detectron2 — the Meta counterpart; cleaner Python API, narrower model catalog, similar academic pedigree. Use when you prefer code over config-inheritance.
- ultralytics/ultralytics — YOLO-family, pip-simple, excellent DX and export tooling. Use when you want a detector shipped, not a research bench (note AGPL licensing).
- open-mmlab/mmyolo — same OpenMMLab idiom, YOLO-specialized. Use when you want MMDet's config style but only YOLO variants.
- huggingface/transformers — has DETR/DETA/RT-DETR detection models with a uniform `AutoModel` API. Use when your stack is already HF and you want transformer detectors only.
- PaddlePaddle/PaddleDetection — comparable breadth on the Paddle framework. Use when you are in the Baidu/Paddle ecosystem rather than PyTorch.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.5.0 | 2018-10 | First public release; COCO 2018 challenge codebase[^1]. |
| 1.0.0 | 2019-08 | First stable line; two-stage + single-stage detectors. |
| 2.0.0 | 2020-05 | Modular refactor; new config/registry conventions[^2]. |
| 2.28 | 2023-01 | Final 2.x maintenance line. |
| 3.0.0 | 2023-04 | Rewrite onto MMEngine; new data structures, config schema[^3]. |
| 3.3.0 | 2024-01 | MM-Grounding-DINO open-vocabulary pipeline added[^5]. |

## References

[^1]: Chen et al., "MMDetection: Open MMLab Detection Toolbox and Benchmark" — arXiv:1906.07155, 2019. https://arxiv.org/abs/1906.07155
[^2]: MMDetection README and model zoo, open-mmlab/mmdetection. https://github.com/open-mmlab/mmdetection
[^3]: MMEngine and MMCV — the runtime and compiled-ops dependencies. https://github.com/open-mmlab/mmengine · https://github.com/open-mmlab/mmcv
[^4]: MMDetection 2.x → 3.x migration guide. https://mmdetection.readthedocs.io/en/latest/migration.html
[^5]: Lyu et al., "RTMDet: An Empirical Study of Designing Real-Time Object Detectors" — arXiv:2212.07784; MM-Grounding-DINO — arXiv:2401.02361. https://arxiv.org/abs/2212.07784

## Tags

python, pytorch, object-detection, instance-segmentation, computer-vision, deep-learning, openmmlab, model-zoo, detr, config-driven
