# CI Suite 速查表

> **suite 名称解析逻辑在 `python/sglang/test/ci/ci_register.py` 的 `effective_suite` 属性中。** 新风格由 `{stage}-test-{runner_config}` 拼接而成；旧风格直接使用 `suite` 字符串。**如拼接逻辑有变化，以 `ci_register.py` 源码为准。**

## CUDA Suite

| Suite | Runner | 典型用途 |
|-------|--------|----------|
| `base-a-test-cpu` | `ubuntu-latest` | CPU-only 单元测试 |
| `base-a-test-1-gpu-small` | `1-gpu-5090` | 快速预检（小模型） |
| `base-b-test-1-gpu-small` | `1-gpu-5090` | 核心 1-GPU 测试（1B 模型） |
| `base-b-test-1-gpu-large` | `1-gpu-h100` | 大内存/Hopper 内核（8B 模型） |
| `base-b-test-2-gpu-large` | `2-gpu-h100` | 2-GPU TP/PP |
| `base-b-test-4-gpu-b200` | `4-gpu-b200` | Blackwell/SM100 早期覆盖 |
| `base-b-kernel-unit-1-gpu-large` | `1-gpu-h100` | JIT 内核正确性 |
| `base-b-kernel-unit-1-gpu-b200` | `1-gpu-b200` | JIT 内核正确性（SM100） |
| `base-b-kernel-unit-8-gpu-h200` | `8-gpu-h200` | 多 GPU JIT 内核正确性 |
| `base-b-kernel-benchmark-1-gpu-large` | `1-gpu-h100` | JIT 内核基准测试 |
| `base-c-test-4-gpu-h100` | `4-gpu-h100` | 大型 4-GPU 集成 |
| `base-c-test-8-gpu-h200` | `8-gpu-h200` | 大型 8-GPU 集成 |
| `base-c-test-deepep-4-gpu-h100` | `4-gpu-h100` | DeepEP 专家并行 |

## AMD Suite

| Suite | 典型用途 |
|-------|----------|
| `stage-a-test-1-gpu-small-amd` | AMD 快速预检 |
| `stage-b-test-1-gpu-small-amd` | AMD 核心 1-GPU 测试 |
| `stage-b-test-1-gpu-large-amd` | AMD 大内存 1-GPU 测试 |
| `stage-b-test-2-gpu-large-amd` | AMD 2-GPU ROCm |
| `stage-c-test-4-gpu-amd` | AMD 4-GPU |
| `stage-c-test-large-8-gpu-amd` | AMD 8-GPU MI325 |

## NPU Suite（Ascend）

| Suite | 典型用途 |
|-------|----------|
| `per-commit-1-npu-a2` | 1-NPU 测试 |
| `per-commit-2-npu-a2` | 2-NPU 测试 |
| `per-commit-4-npu-a3` | 4-NPU 测试 |
| `per-commit-16-npu-a3` | 16-NPU 测试 |
| `multimodal-gen-test-1-npu-a3` | 1-NPU 多模态 |
| `multimodal-gen-test-2-npu-a3` | 2-NPU 多模态 |
| `multimodal-gen-test-8-npu-a3` | 8-NPU 多模态 |

> **NPU suite 后缀为硬件型号**（`a2` = Ascend 910B，`a3` = Ascend 910C），不是 `small`/`large` 等通用级别。
