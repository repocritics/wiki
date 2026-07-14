# ultralytics/ultralytics

> The YOLO framework: one PyTorch package for training, validating, exporting, and deploying object detection, segmentation, pose, and classification models — under AGPL-3.0.

[GitHub repo](https://github.com/ultralytics/ultralytics) ·
[Official website](https://platform.ultralytics.com) ·
[License: AGPL-3.0](https://github.com/ultralytics/ultralytics/blob/main/LICENSE)

## Overview

`ultralytics` is the successor to the widely-copied `yolov5` repo. It packages the entire computer-vision workflow — dataset loading, augmentation, training, validation, tracking, and export — behind a single `YOLO` class and a `yolo` CLI[^1]. First released as the "YOLOv8" package in January 2023, it has since become the default on-ramp for practitioners who want a working detector without assembling a research codebase. The repo bundles multiple model generations (YOLOv3 through the current YOLO26) plus integrated third-party architectures (RT-DETR, SAM, YOLO-World), all reachable through the same API.

The name "YOLO" carries lineage that Ultralytics did not originate. The first three versions came from Joseph Redmon's Darknet; v4 and v7 came from Alexey Bochkovskiy and the Wang et al. group. Ultralytics' YOLOv5 (2020) reimplemented the idea in PyTorch and adopted the brand without a companion paper, which drew persistent criticism about versioning that outpaces peer review[^2]. The models are genuinely fast and easy to use; the marketing "state-of-the-art" claims are best treated as vendor benchmarks (see Production Notes).

The defining tension is the **license**. `ultralytics` is AGPL-3.0, and so are the pretrained weights the package auto-downloads on first use. AGPL's network-copyleft clause reaches SaaS deployments, not just redistributed binaries — meaning most commercial products that embed a YOLO model either buy an Ultralytics Enterprise License or must open-source their application[^3]. This single fact governs whether the project is usable for a given team more than any accuracy number.

## Getting Started

```bash
pip install ultralytics        # Python >= 3.8, PyTorch >= 1.8
```

```python
from ultralytics import YOLO

model = YOLO("yolo26n.pt")     # auto-downloads weights on first use

# Train on a small bundled dataset
model.train(data="coco8.yaml", epochs=100, imgsz=640, device="cpu")

# Inference
results = model("path/to/image.jpg")
results[0].show()

# Export for deployment
model.export(format="onnx")    # also: engine, coreml, tflite, openvino
```

```bash
# Same functionality from the CLI
yolo predict model=yolo26n.pt source='https://ultralytics.com/images/bus.jpg'
yolo train model=yolo26n.pt data=coco8.yaml epochs=100 imgsz=640
```

Model suffixes encode scale (`n`/`s`/`m`/`l`/`x`) and task (`-seg`, `-pose`, `-cls`, `-obb`, `-sem`). Weights download automatically from the `ultralytics/assets` releases.

## Architecture / How It Works

The package is organized around a **task-and-mode matrix**. Tasks are detect, segment, classify, pose, oriented bounding box (OBB), and semantic segmentation; modes are train, val, predict, export, track, and benchmark. The `YOLO` class dispatches to task-specific trainer/validator/predictor classes, so `model.train()` and `model.predict()` share one entry point regardless of task.

Models are defined by **YAML architecture files** (e.g. `yolo26.yaml`), not hand-written `nn.Module` subclasses. A parser reads the YAML, resolves module names (`Conv`, `C2f`, `SPPF`, detection `Head`) against a registry, and constructs the network with a width/depth multiplier per scale. This is why one YAML yields the entire `n`→`x` family and why custom architectures are edited as config rather than code.

The detection head has been **anchor-free** since YOLOv8, using distribution-focal-loss regression instead of predefined anchor boxes[^1]. Newer generations move toward **end-to-end / NMS-free** inference: the repo reports separate `mAP(e2e)` metrics for YOLO26, indicating a variant that removes the non-maximum-suppression post-processing step to simplify deployment[^4]. Whether NMS-free is enabled affects both latency and the exported graph.

**Export** is a first-class subsystem. `model.export()` traces the PyTorch graph and emits ONNX, TensorRT engines, CoreML, TFLite, OpenVINO, TorchScript, PaddlePaddle, and others. Each format has its own optional heavy dependency (`onnx`, `tensorrt`, `coremltools`), installed on demand. The exported artifact is where most production deployments actually run — the PyTorch path is primarily for training.

**Tracking** (ByteTrack, BoT-SORT) is layered on top of the detector as a stateful wrapper, not a separate model, and works with any detect/segment/pose checkpoint.

## Production Notes

**Licensing is the first design decision, not an afterthought.** AGPL-3.0 applies to the code and the pretrained weights. If your product exposes YOLO output over a network — an API, a web app, an internal tool served to users — the copyleft obligation can attach to your whole application. The commercial escape hatch is an Ultralytics Enterprise License (paid, per-negotiation)[^3]. Teams that cannot accept either option train from randomly-initialized weights on permissively-licensed architectures instead, or switch frameworks (see Alternatives).

**Telemetry is on by default.** The package collects anonymized usage analytics (via Sentry and Google Analytics in prior versions) and syncs settings. Disable with `yolo settings sync=False` or by editing the settings file; in restricted/offline environments this matters for compliance review[^5].

**Benchmarks are vendor-reported.** The README's speed and mAP tables are measured on Ultralytics' own hardware (e.g. an AWS P4d for GPU, ONNX runtime for CPU). They are internally consistent and useful for comparing scales, but the "SOTA" framing compares favorably-chosen baselines. Reproduce numbers on your own hardware and data before committing to a model size.

**Auto-download is a supply-chain and reproducibility surface.** First inference silently fetches weights over the network. In air-gapped or CI environments, pre-stage the `.pt` files and pin the `ultralytics` version — the default `main`-tracking install can change model defaults between releases.

**Version churn.** The package uses a single `8.x` version line that spans multiple model generations, so `pip install ultralytics` can quietly change which model `yolo26n.pt` resolves to or alter training defaults. Pin an exact version in `requirements.txt` for any reproducible pipeline.

**Export ≠ parity.** Numerical results can drift between the PyTorch model and its ONNX/TensorRT export (different NMS implementations, FP16 quantization, dynamic vs. static shapes). Validate the exported artifact with `yolo val`, not just the training checkpoint.

## When to Use / When Not

**Use when:**
- You need a working detector/segmenter/pose model fast, with training and export in one package.
- Your use is research, internal, open-source (AGPL-compatible), or covered by a purchased Enterprise License.
- You want a single API across detection, segmentation, classification, pose, OBB, and tracking.
- You need broad export targets (TensorRT/CoreML/TFLite/OpenVINO) without wiring them yourself.

**Avoid when:**
- You're shipping closed-source commercial software and won't buy the Enterprise License — AGPL is a hard blocker.
- You need peer-reviewed, citable architectures for academic work (the versioning is product-driven, not paper-driven).
- You want a permissively-licensed detection stack you can vendor freely (use torchvision/Detectron2/MMDetection).
- You only need inference post-processing/annotation and already have a model — a lighter utility library fits better.

## Alternatives

- roboflow/supervision — MIT-licensed detection utilities (annotation, tracking, metrics); use when you have a model and want permissive post-processing rather than a full framework.
- open-mmlab/mmdetection — Apache-2.0 research zoo with dozens of architectures; use when you need reproducible academic baselines and don't want AGPL.
- facebookresearch/detectron2 — Apache-2.0 detection/segmentation framework from Meta; use when you want a mature, permissively-licensed research codebase.
- WongKinYiu/yolov9 — the "official" YOLOv9 lineage from the v4/v7 authors; use when you want the Wang-group models under their own terms.
- PaddlePaddle/PaddleDetection — Apache-2.0 detection suite including PP-YOLOE; use when you want strong models on a non-AGPL stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| YOLOv5 | 2020-06 | Ultralytics' first PyTorch YOLO, in the separate `yolov5` repo; adopted the YOLO brand without a paper[^2]. |
| 8.0 (YOLOv8) | 2023-01 | New unified `ultralytics` package, `YOLO` API + `yolo` CLI, anchor-free head[^1]. |
| 8.1–8.2 | 2024 | Integrated RT-DETR, SAM/FastSAM, YOLO-World; added OBB task. |
| 8.3 (YOLO11) | 2024-09 | Announced at YOLO Vision 2024; renamed from the expected "YOLOv11" to "YOLO11". |
| 8.4 (YOLO26) | 2026 | Current generation; end-to-end / NMS-free variants, added semantic-segmentation task[^4]. |

## References

[^1]: Ultralytics Docs — YOLOv8 and the unified package/CLI. https://docs.ultralytics.com/
[^2]: Discussion of YOLO version lineage and the naming of YOLOv5/YOLOv8 without accompanying papers. https://docs.ultralytics.com/models/yolov5/
[^3]: Ultralytics licensing — AGPL-3.0 vs. Enterprise License. https://www.ultralytics.com/license
[^4]: Repository README, model tables reporting `mAP(e2e)` for YOLO26 (end-to-end / NMS-free). https://github.com/ultralytics/ultralytics
[^5]: Ultralytics settings and analytics — `yolo settings sync=False`. https://docs.ultralytics.com/quickstart/#ultralytics-settings

## Tags

python, pytorch, computer-vision, object-detection, yolo, instance-segmentation, pose-estimation, deep-learning, onnx-export, edge-inference, agpl
