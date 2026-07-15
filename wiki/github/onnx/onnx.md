# onnx/onnx

> An open, framework-neutral file format for machine learning models — a protobuf schema plus a versioned operator catalog, not a runtime.

[GitHub repo](https://github.com/onnx/onnx) ·
[Official website](https://onnx.ai/) ·
[License: Apache-2.0](https://github.com/onnx/onnx/blob/main/LICENSE)

## Overview

ONNX (Open Neural Network Exchange) is a specification for representing machine
learning models as a serialized computation graph, together with the reference
tooling that reads, writes, checks, and infers shapes over that format. It was
announced in September 2017 by Microsoft and Facebook as a way to move a trained
model out of the framework it was authored in (PyTorch, TensorFlow, scikit-learn,
Keras) and into a different framework, runtime, or hardware backend for
inference[^1]. It became a Linux Foundation / LF AI & Data project in 2019,
which is why governance runs through SIGs, working groups, and a steering
committee rather than a single vendor[^2].

The single most important thing to understand is what ONNX is *not*: it is not an
inference engine. `pip install onnx` gives you the format, a model checker, shape
inference, and a (deliberately slow) Python reference implementation of the
operators — it does not give you fast execution. Fast execution comes from a
separate runtime such as ONNX Runtime, TensorRT, OpenVINO, or a mobile engine
that consumes the `.onnx` file. Conflating `onnx` with `onnxruntime` is the most
common source of confusion in the ecosystem.

The defining tension is **portability versus operator coverage**. ONNX only
guarantees interoperability for the operators in its standard catalog at a given
*opset* (operator-set) version. Any model that uses a framework feature outside
that catalog — a custom op, exotic control flow, an op newer than the target
runtime supports — either fails to export or exports to something no runtime can
run. The format is stable and well-specified; getting a real-world model *into*
it cleanly is where the work is.

## Getting Started

```sh
pip install onnx                 # format + checker + shape inference
pip install onnx[reference]      # optional pure-Python reference operators
```

Most users never construct ONNX by hand — they export from a framework. The
`onnx` package is what you use to inspect, validate, and post-process the result:

```python
import onnx

model = onnx.load("model.onnx")

# Structural validation: opset compatibility, dangling inputs, type errors
onnx.checker.check_model(model)

# Propagate tensor shapes/types through the graph (best-effort)
model = onnx.shape_inference.infer_shapes(model)

print(onnx.helper.printable_graph(model.graph))
print("IR version:", model.ir_version)
print("opset:", [(o.domain or "ai.onnx", o.version) for o in model.opset_import])
```

Exporting from PyTorch (the common real path) uses the framework's exporter, not
`onnx` directly:

```python
import torch
torch.onnx.export(net, sample_input, "model.onnx", opset_version=17)
```

## Architecture / How It Works

An ONNX model is a **Protocol Buffers message** (`ModelProto`) defined in
`onnx.proto`[^3]. The core objects:

- **`GraphProto`** — a dataflow graph: a list of `NodeProto` operations,
  `TensorProto` initializers (weights), and typed graph inputs/outputs. The graph
  is a DAG; loops and conditionals are expressed with the `Loop`, `Scan`, and
  `If` operators whose bodies are nested subgraphs.
- **Operators** — each `NodeProto` names an op (`Conv`, `MatMul`, `Gather`, …) in
  a **domain**. The base domain is `ai.onnx`; `ai.onnx.ml` holds classical-ML ops
  (tree ensembles, linear models) for the scikit-learn/XGBoost path.
- **Two version numbers that are easy to confuse**: the **IR version** versions
  the protobuf schema itself (rarely changes); the **opset version** versions the
  operator semantics and is what actually matters for compatibility. A model
  imports one opset per domain.

The repository ships three things beyond the schema: the **checker** (validates a
model is well-formed for its declared opset), **shape and type inference** (a
static pass that annotates value shapes — incomplete for some ops, and it never
executes the graph), and the **version converter** (rewrites a graph between
opsets, with per-op adapters that do not all exist). Operator definitions live as
C++ `OpSchema` registrations with generated docs in `docs/Operators.md`; adding
or changing an op is a governed process with its own contribution document[^4].

Backward compatibility is one-directional by design: opset numbers only increase,
old operator versions are retained, and a runtime advertises the opset *range* it
supports. This is what lets a model exported years ago still load, provided the
consuming runtime kept the old op versions.

## Production Notes

**Opset drift is the real operational cost.** A model exported at opset 18 will
not run on a runtime that only implements up to opset 15, and a model at opset 11
may hit deprecated-op behavior on a newer engine. In practice you pin the exporter
`opset_version` to the lowest version your *target runtime* fully supports, not
the newest ONNX offers. The `onnx.version_converter` can bridge gaps but silently
lacks adapters for some ops — verify, don't assume.

**The 2GB protobuf ceiling.** Protobuf caps a single message at 2GB, so any model
whose weights exceed that (most modern LLMs, large diffusion models) must use the
**external-data** format, where `TensorProto` weights are written to sidecar files
and the `.onnx` holds only references. Tooling that loads with
`load_external_data=False`, or moves the `.onnx` without its sidecar files, breaks
in confusing ways. Treat an ONNX export of a large model as a *directory*, not a
file.

**Export failures dominate over runtime failures.** The hard part is almost always
the framework exporter, not ONNX. Dynamic shapes, Python-side control flow that
should be traced, unsupported ops, and custom CUDA kernels all produce either an
export error or — worse — a silently wrong graph. PyTorch's `torch.onnx.export`
(TorchScript tracer) and the newer `dynamo`-based exporter can produce different
graphs for the same model. Always run `checker.check_model` and compare a few
outputs against the source framework before trusting an export.

**Shape inference is best-effort.** `infer_shapes` will leave shapes unknown for
ops it cannot reason about, and downstream tooling that assumes fully-inferred
shapes will fail on those graphs. For models over 2GB use
`onnx.shape_inference.infer_shapes_path` (operates on files, not in-memory
protos).

**Quantization has two incompatible spellings.** Quantized models appear either as
`QuantizeLinear`/`DequantizeLinear` (QDQ) nodes around normal ops, or as fused
`QLinearConv`-style operators (QOperator). Different runtimes optimize different
representations; picking the wrong one leaves performance on the table.

**Training is effectively dormant.** ONNX defined a training extension, but the
ecosystem's energy is entirely on inference. Do not plan an ONNX-based training
pipeline expecting broad support.

## When to Use / When Not

**Use when:**
- You need to decouple where a model is *trained* from where it is *served*
  (train in PyTorch, deploy via TensorRT / OpenVINO / a mobile or web runtime).
- You want a single artifact that many hardware vendors' toolchains can ingest.
- You're shipping classical ML (scikit-learn, XGBoost, LightGBM) to a non-Python
  serving environment via `ai.onnx.ml`.
- You need a stable, inspectable, vendor-neutral archival format for a model.

**Avoid when:**
- You train and serve in the same framework — an in-framework export (TorchScript,
  SavedModel, `torch.compile`) skips a lossy conversion step.
- Your model leans on custom ops or dynamic control flow the exporter can't trace;
  the export effort may exceed the interoperability benefit.
- You expected a runtime — reach for onnxruntime, not onnx, if you want to *run*
  the model.
- You need training portability rather than inference portability.

## Alternatives

- microsoft/onnxruntime — the runtime, not a substitute for the format; use it
  *with* onnx when you actually need to execute a model.
- openxla/stablehlo — MLIR-dialect model IR gaining traction as a compiler-facing
  exchange format; use it when your target is an XLA/MLIR compiler stack.
- pytorch/executorch — use instead when you're staying in the PyTorch world and
  targeting on-device inference without the ONNX round-trip.
- apple/coremltools — use instead when Apple platforms are the only deployment
  target and Core ML's optimizations matter more than portability.
- tensorflow/tensorflow (SavedModel/TFLite) — use instead when you're end-to-end
  in the TensorFlow ecosystem and don't need to leave it.

## History

| Version | Date | Notes |
|---------|------|-------|
| announced | 2017-09 | Introduced by Microsoft and Facebook[^1]. |
| 1.0 | 2017-12 | First tagged release of the format and tooling. |
| — | 2019-11 | Joined the Linux Foundation (LF AI & Data)[^2]. |
| 1.10 | 2021 | opset 15; external-data and shape-inference tooling matured. |
| 1.13 | 2022–23 | opset 18; classical-ML and quantization ops expanded. |
| 1.16 | 2024 | opset 21. |
| 1.17 | 2024 | opset 22; abi3 wheels, reproducible Linux builds. |

## References

[^1]: Microsoft, "Microsoft and Facebook create open ecosystem for AI model interoperability" — 2017-09-07. https://azure.microsoft.com/en-us/blog/microsoft-and-facebook-create-open-ecosystem-for-ai-model-interoperability/
[^2]: LF AI & Data Foundation — ONNX project page. https://lfaidata.foundation/projects/onnx/
[^3]: ONNX intermediate representation spec (IR.md). https://github.com/onnx/onnx/blob/main/docs/IR.md
[^4]: ONNX operator documentation (Operators.md) and "Add a new operator" guide. https://github.com/onnx/onnx/blob/main/docs/Operators.md

## Tags

python, machine-learning, deep-learning, model-format, interoperability, protobuf, inference, neural-network, onnx, mlops, linux-foundation
