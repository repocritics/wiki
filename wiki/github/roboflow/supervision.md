# roboflow/supervision

> Model-agnostic computer-vision utilities — the post-processing, annotation, tracking, and dataset glue that sits between a detection model and your application.

[GitHub repo](https://github.com/roboflow/supervision) ·
[Official website](https://supervision.roboflow.com) ·
[License: MIT](https://github.com/roboflow/supervision/blob/develop/LICENSE.md)

## Overview

Supervision is a Python library from Roboflow, first published in late 2022[^1]. It deliberately does not train or run models. Instead it owns everything that happens around inference: normalizing model outputs into one data structure, drawing boxes/masks/labels/traces onto frames, multi-object tracking, zone and line-crossing counting, dataset loading/conversion, and detection metrics. The pitch is that every computer-vision project re-implements this same glue badly, so it should be a maintained dependency instead.

The defining design choice is `sv.Detections` — a single NumPy-backed container (boxes, masks, confidence, class IDs, tracker IDs, plus an arbitrary `data` dict) that every other component consumes and produces. Model connectors (`from_ultralytics`, `from_inference`, `from_transformers`, `from_detectron2`, and others) exist only to map a specific framework's output into this schema; from that point the annotators, trackers, zones, and metrics are model-independent. This is the library's strength (uniform interface across YOLO, RF-DETR, RT-DETR, SAM, Transformers) and its coupling story: if your model's output does not map cleanly onto the `Detections` fields, you are writing an adapter, and features you want are only as good as their support for that one structure.

The tradeoff worth naming up front: Supervision moved fast through its `0.x` series and made real breaking API changes between minor versions. It is widely used and well-documented, but tutorials and Stack Overflow answers written against older versions frequently no longer run.

## Getting Started

```bash
pip install supervision   # Python >= 3.10
```

```python
import cv2
import supervision as sv
from ultralytics import YOLO

model = YOLO("yolo11n.pt")
image = cv2.imread("image.jpg")

result = model(image)[0]
detections = sv.Detections.from_ultralytics(result)   # normalize into sv.Detections

box_annotator = sv.BoxAnnotator()
label_annotator = sv.LabelAnnotator()

annotated = box_annotator.annotate(scene=image.copy(), detections=detections)
annotated = label_annotator.annotate(scene=annotated, detections=detections)

cv2.imwrite("out.jpg", annotated)
```

Note `image.copy()`: annotators draw in place and mutate the scene you pass unless you copy it first.

## Architecture / How It Works

Everything orbits `sv.Detections`. Internally it is a set of parallel NumPy arrays indexed by detection: `xyxy` (absolute pixel corners, not normalized), optional `mask`, `confidence`, `class_id`, `tracker_id`, and a `data` dict for anything else (class names, keypoints, custom fields). It supports NumPy-style indexing and boolean filtering (`detections[detections.confidence > 0.5]`, `detections[detections.class_id == 0]`), merging, and `with_nms()` / `with_nmm()` for class-aware or class-agnostic suppression.

Around that core:

- **Connectors** — `Detections.from_*` classmethods for Ultralytics, Roboflow Inference, Hugging Face Transformers, MMDetection, Detectron2, YOLO-NAS, PaddleDetection, SAM/SAM2, and others. Some third-party models (e.g. `rfdetr`) return `sv.Detections` directly.
- **Annotators** — one class per visual style (`BoxAnnotator`, `RoundBoxAnnotator`, `MaskAnnotator`, `PolygonAnnotator`, `LabelAnnotator`, `RichLabelAnnotator`, `HeatMapAnnotator`, `TraceAnnotator`, `BlurAnnotator`, `PixelateAnnotator`, `EllipseAnnotator`, and more). They compose by chaining `annotate()` calls over the same scene.
- **Trackers** — `sv.ByteTrack` is the primary integrated tracker; it assigns `tracker_id` given per-frame detections. `DetectionsSmoother` reduces jitter across frames.
- **Spatial tools** — `PolygonZone` (count detections inside a region), `LineZone` (count crossings over a line, with in/out direction), and `InferenceSlicer` (SAHI-style slicing-aided inference for small objects, with overlap and NMS merging).
- **Datasets** — `DetectionsDataset` / `DetectionDataset` and `ClassificationDataset` load and export COCO, YOLO, and Pascal VOC, with lazy per-image loading, `split()`, and `merge()`.
- **Metrics** — mean average precision, precision/recall/F1, and `ConfusionMatrix`, computed from `Detections` pairs.

The base is NumPy and OpenCV/Pillow; PyTorch is not a core dependency, which keeps the install light. Model frameworks are your problem to install — Supervision only touches their outputs.

## Production Notes

**Coordinate conventions bite.** `Detections.xyxy` is absolute pixel coordinates. Model outputs that are normalized or in `xywh` (YOLO's native format) must be converted before constructing `Detections` by hand; the `from_*` connectors handle this, hand-rolled code often does not.

**Annotators mutate in place.** `annotate(scene=frame, ...)` draws onto `frame`. In a video loop, reusing the same buffer without `.copy()` accumulates overdraw. This is the single most common beginner bug.

**The annotator API was reorganized.** Earlier versions had a `BoxAnnotator` that also drew labels; labeling was later split into `LabelAnnotator`, and the box annotator's label parameters were removed[^2]. Any tutorial calling `BoxAnnotator(...).annotate(labels=...)` predates the split and will not run on current releases. Pin your version and read the docs for that version — the site is versioned (`/latest/` vs `/develop/`).

**Tracking is stateful and order-sensitive.** `ByteTrack.update_with_detections()` must be called once per frame in order; feed frames out of order and IDs scramble. Untracked detections carry `tracker_id = None`, so filtering on `tracker_id` needs a null check.

**Slicing costs latency and memory.** `InferenceSlicer` runs the model on each tile, so throughput drops roughly with tile count; tune slice size, overlap, and the merge NMS threshold rather than accepting defaults. It is the right tool for small-object detection on large frames and the wrong tool for real-time on a single object scale.

**Version churn.** The metrics module was reworked (older `sv.MeanAveragePrecision` gave way to a new `metrics` package), keypoint/vertex annotators and SAM2 support arrived over the 2024 releases, and the minimum Python was raised to 3.10. Treat a Supervision version bump as a potential breaking change and read the changelog, not just the semver.

## When to Use / When Not

**Use when:**
- You have a detection/segmentation/keypoint model and need annotation, tracking, zone counting, or metrics without writing them yourself.
- You want to stay model-agnostic and swap backbones (YOLO ↔ RF-DETR ↔ Transformers) behind one data structure.
- You need dataset format conversion (COCO ↔ YOLO ↔ Pascal VOC) as a utility.
- You are prototyping a real-time video analytics pipeline (dwell time, line counting, speed estimation).

**Avoid when:**
- You need training, model weights, or an inference engine — Supervision provides none; pair it with Ultralytics, Inference, or Transformers.
- You are locked into one framework that already ships equivalent visualization/tracking (e.g. Ultralytics' built-in plotting) and don't need the abstraction.
- Your output schema doesn't fit the `Detections` fields (exotic 3D, panoptic, or graph outputs) and the adapter cost exceeds the benefit.

## Alternatives

- ultralytics/ultralytics — use instead when you are all-in on YOLO and want built-in plotting/tracking without a second dependency.
- obss/sahi — use instead when slicing-aided inference for small objects is your whole problem; Supervision's `InferenceSlicer` overlaps it but SAHI is more focused.
- voxel51/fiftyone — use instead when the job is dataset curation, exploration, and evaluation at scale rather than per-frame annotation.
- open-mmlab/mmdetection — use instead when you want a full training+inference framework with its own visualization, not a post-processing add-on.
- albumentations/albumentations — use instead for augmentation; adjacent CV utility, different stage of the pipeline.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2022-11 | Initial release; `Detections` + basic annotators[^1]. |
| 0.16.0 | 2023-11 | Annotator API redesign; labeling split into `LabelAnnotator`[^2]. |
| 0.18.0 | 2024-01 | `LineZone` / `PolygonZone` refinements, smoothing utilities. |
| 0.22.0 | 2024-07 | Reworked `metrics` module; keypoint support; SAM2 connector. |
| 0.25.0 | 2025 | Continued annotator/metrics additions; Python >= 3.10 baseline. |

Exact per-release dates and contents are in the GitHub releases and changelog[^3]; treat the above as the shape of the evolution, not a citation of specifics.

## References

[^1]: roboflow/supervision, PyPI release history. https://pypi.org/project/supervision/#history
[^2]: Supervision annotators documentation (versioned). https://supervision.roboflow.com/latest/detection/annotators/
[^3]: Supervision releases / changelog. https://github.com/roboflow/supervision/releases

## Tags

python, computer-vision, object-detection, image-annotation, object-tracking, video-processing, deep-learning, dataset-tools, yolo, roboflow, machine-learning
