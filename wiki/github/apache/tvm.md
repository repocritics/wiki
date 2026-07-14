# apache/tvm

> An end-to-end compiler stack that lowers trained ML models to optimized code for CPUs, GPUs, and accelerators — with the optimization passes exposed and editable in Python.

[GitHub repo](https://github.com/apache/tvm) ·
[Official website](https://tvm.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/tvm/blob/main/LICENSE)

## Overview

TVM is a machine-learning compiler framework: it takes a model from a training framework (PyTorch, ONNX, TensorFlow) and compiles it down to a minimal, self-contained deployable module targeting a specific backend — LLVM CPU code, CUDA, ROCm, Metal, OpenCL, Vulkan/SPIR-V, or WebGPU/WASM. It began as a research project at the University of Washington (Tianqi Chen et al., ~2017) and entered the Apache Software Foundation incubator in 2019, graduating as a top-level project in 2020[^1].

The defining bet is **Python-first, cross-level compilation**. Where most inference stacks hide their optimizer behind a fixed C++ pipeline, TVM exposes the intermediate representations and the transformation passes as manipulable objects you can write and reorder in Python. The current design splits into two levels: **Relax** (graph/computational-graph IR, successor to the older Relay) and **TensorIR / TIR** (tensor-program IR, where loop tiling, vectorization, and memory scope live)[^2]. The stated goal is to let a compiler engineer jointly optimize across the graph, the tensor loops, and hand-written library calls in one framework.

That flexibility is also the cost. TVM is not a drop-in `model.compile()` box; it is a toolkit for people building inference pipelines. The most visible downstream consumer is MLC-LLM, which uses the Relax flow to run LLMs on phones, browsers, and consumer GPUs — a good illustration of what TVM is for and how much surrounding code a real deployment needs.

## Getting Started

```bash
# Prebuilt nightly wheels (CPU + CUDA variants published by the project)
pip install apache-tvm --pre
# Or build from source for a specific backend (LLVM required):
#   git clone --recursive https://github.com/apache/tvm
#   cmake + set(USE_LLVM ...), set(USE_CUDA ...) in config.cmake
```

```python
import tvm
from tvm import relax
from tvm.relax.frontend.torch import from_exported_program
import torch, torch.nn as nn

class MLP(nn.Module):
    def __init__(self): super().__init__(); self.fc = nn.Linear(128, 10)
    def forward(self, x): return self.fc(x).relu()

ep = torch.export.export(MLP().eval(), (torch.randn(1, 128),))
mod = from_exported_program(ep)                      # -> Relax IRModule

target = tvm.target.Target("llvm -num-cores=8")      # or "cuda", "metal", ...
ex = relax.build(mod, target)                        # compile
vm = relax.VirtualMachine(ex, tvm.cpu())
out = vm["main"](tvm.nd.array(torch.randn(1, 128).numpy()))
```

The exact frontend entry points move between releases; always check the docs for the version you installed rather than copying tutorials wholesale[^3].

## Architecture / How It Works

The compile flow is a lowering pipeline across two IR levels:

1. **Import** — a model is captured into a Relax `IRModule` via a frontend (`torch.export`, ONNX, or the Relax NN builder). The IRModule is the unit everything operates on; it holds both graph-level (`relax`) functions and tensor-level (`tir`) functions in the same container.
2. **Relax graph passes** — operator fusion, layout transforms, constant folding, and dispatch. Relax replaced Relay as the default graph IR; Relax is dynamic-shape-aware (symbolic shape variables are first-class), which Relay handled poorly.
3. **TensorIR (TIR)** — each compute-heavy operator lowers to a loop nest in TIR. This is where scheduling happens: tiling, vectorization, thread binding, cache-read/write, and memory scopes. TIR schedules are written as explicit transformations on the loop program.
4. **Codegen** — TIR lowers to LLVM IR, source-level CUDA/Metal/OpenCL, or SPIR-V, producing a runtime module. The lightweight C++ **TVM runtime** loads that module and executes it via the Relax VM.

**How the "optimal schedule" gets chosen** is TVM's signature problem, and it has three historical answers, each superseding the last[^4]:
- **AutoTVM** — template-based; a human writes a schedule template with tunable knobs, TVM searches the knob space with a cost model.
- **Ansor / auto-scheduler** — removes the hand-written template; generates the search space automatically.
- **MetaSchedule** — the current unified probabilistic-search framework, and the one wired into the Relax flow.

**BYOC (Bring Your Own Codegen)** lets you offload subgraphs to an external library or accelerator backend (cuDNN, CUTLASS, TensorRT, a custom NPU) while TVM compiles the rest — the mechanism vendors use to integrate hardware without forking the whole stack.

## Production Notes

**Tuning is a real, expensive step, not a flag.** Getting competitive numbers out of TVM historically meant running MetaSchedule/AutoTVM auto-tuning for minutes to hours per workload, per target shape, on the target hardware. The tuning log is an artifact you must generate, store, and version. Skip it and untuned kernels can be far slower than vendor libraries; the payoff comes from tuned kernels plus fusion on shapes the vendor libraries don't specialize for.

**Build-from-source is the common path and it is heavy.** Backend support (CUDA, ROCm, Vulkan, Metal) is a compile-time `config.cmake` decision, and LLVM is effectively required. Prebuilt wheels exist but may not carry the exact backend/CUDA-version combination you need, so most nontrivial users compile against their own LLVM. This is the first real friction point for newcomers.

**API churn is the biggest maintenance tax.** TVM has been through multiple ground-up redesigns — NNVM → Relay → Relax at the graph level, and the "TVM Unity" reorganization that made Relax + TensorIR the default. Relay is legacy; tutorials and Stack Overflow answers from before the Relax transition often no longer apply, and pinning a TVM version is essential because IR-level APIs are not stable across minor releases.

**Deployment target is a `runtime`, not a service.** TVM outputs a compiled module plus a small C++ runtime you embed; it does not ship a serving layer, batching, or a model server. You build that around it (or adopt something like MLC-LLM that already has).

**Dynamic shapes are supported but sharp.** Relax's symbolic shapes make LLM/variable-sequence workloads possible, but shape-dependent dispatch and tuning across a range of shapes remain areas where you should expect to read source and profile, not trust defaults.

## When to Use / When Not

**Use when:**
- You need to deploy the *same* model across heterogeneous hardware (CPU + several GPU vendors + WebGPU) from one toolchain.
- You are targeting an accelerator or edge device with no first-class vendor runtime and need BYOC or custom codegen.
- You want to research or customize compiler passes — the Python-exposed IR is the reason to be here.
- You have shapes or fused patterns that vendor libraries (cuDNN/oneDNN/TensorRT) don't optimize well, and tuning time is acceptable.

**Avoid when:**
- You just want fast inference of a standard model on NVIDIA — TensorRT or vLLM will get you there with far less work.
- You need a serving stack (batching, multi-model, autoscaling) out of the box.
- You can't budget the auto-tuning and build-from-source effort, or you need API stability across upgrades.
- Your model is a mainstream LLM and you'd be better served by MLC-LLM (which is built *on* TVM) than by wiring Relax yourself.

## Alternatives

- openai/triton — Python DSL for writing GPU kernels directly; lower-level and NVIDIA-centric, no end-to-end graph compilation. Use it when you're hand-authoring a few hot kernels rather than compiling whole models.
- pytorch/pytorch (torch.compile / Inductor) — in-framework compilation; use when you live inside PyTorch and want speedups without leaving it.
- NVIDIA/TensorRT — best-in-class on NVIDIA GPUs with minimal effort; use when you only target NVIDIA and don't need portability.
- openxla/xla (and IREE) — Google's ML compiler lineage (MLIR-based); use when you're in the JAX/TF world or want an MLIR-native stack.
- mlc-ai/mlc-llm — built on TVM's Relax flow; use when your specific goal is LLMs on edge/consumer devices rather than a general compiler.

## History

| Version | Date | Notes |
|---------|------|-------|
| Research release | 2017 | UW SAMPL project; NNVM graph IR + TVM tensor IR[^1]. |
| Apache incubation | 2019-03 | Entered ASF incubator. |
| Relay | 2019 | Second-gen graph IR, replaced NNVM. |
| Top-level project | 2020-11 | Graduated ASF incubator[^1]. |
| 0.8 | 2021-11 | AutoScheduler (Ansor) + MetaSchedule maturing; first ASF top-level release line. |
| TVM Unity / Relax | 2023 | Relax + TensorIR cross-level flow becomes the forward direction[^2]. |
| 0.2x line | 2024–2026 | Relax default; PyTorch `torch.export` frontend; Relay legacy. |

(Version-to-date mapping for point releases is approximate; consult the release notes for exact tags[^5].)

## References

[^1]: Tianqi Chen et al., "TVM: An Automated End-to-End Optimizing Compiler for Deep Learning," OSDI 2018. https://www.usenix.org/conference/osdi18/presentation/chen
[^2]: TVM Unity / Relax design overview. https://tvm.apache.org/docs/arch/index.html
[^3]: TVM Documentation — Getting Started. https://tvm.apache.org/docs/get_started/overview.html
[^4]: MetaSchedule / auto-tuning documentation. https://tvm.apache.org/docs/how_to/tune_with_autotvm/index.html
[^5]: TVM release notes. https://github.com/apache/tvm/blob/main/NEWS.md

## Tags

machine-learning, compiler, deep-learning, tensor, gpu, cuda, vulkan, code-generation, model-inference, python, apache, mlir-alternative
