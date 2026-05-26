# 测试检视检查清单

适用于 `test/`、`python/sglang/jit_kernel/tests/`、`python/sglang/jit_kernel/benchmark/`、`sgl-model-gateway/tests/`、`sgl-model-gateway/e2e_test/` 及所有新增测试文件。

> **作用域限制：** 本清单仅覆盖 `test/`、`python/`、`sgl-model-gateway/` 目录内的测试。`benchmark/` 顶层目录不在检视范围内。
> **最后验证：** 2026-05（与 `ci_register.py` 签名、`CustomTestCase` 位置同步校验）

---

## 1. CI 注册（所有测试必须通过）

每个 CI 自动发现的测试文件必须满足：

- [ ] `register_*_ci(...)` 在**模块级**调用（不在函数或条件内）
- [ ] `est_time` 是**字面数值**（`int` 或 `float`，不是变量）— `ci_register.py` 通过 AST 解析，非常量表达式会被拒绝
- [ ] `suite` 与 `stage`+`runner_config` 二选一（互斥）：
  - **旧风格**：`suite="base-b-test-1-gpu-large"`（直接指定完整 suite 名称）
  - **新风格**：`stage="base-b", runner_config="1-gpu-large"`（由 `ci_register.py` 的 `effective_suite` 属性拼接为 `"base-b-test-1-gpu-large"`）
  - 两种风格均合法，同一文件内保持一致即可
- [ ] `est_time` 切合实际（本地测量过，留有 CI 波动余量）
- [ ] 无不必要的 `register_amd_ci` / `register_npu_ci` / `register_xpu_ci`（仅当测试后端特定代码路径时才添加）
- [ ] 被禁用的测试使用 `disabled="reason"` 并附带 issue 追踪编号

### 注册函数一览

所有注册函数均定义在 `python/sglang/test/ci/ci_register.py`。除 `register_xpu_ci` 外，其余函数签名为 `(est_time, suite=None, nightly=False, disabled=None, *, stage=None, runner_config=None)`。`register_xpu_ci` 仅接受 `(est_time, suite, nightly=False, disabled=None)`，**不支持** `stage`/`runner_config`。

| 函数 | 用途 |
|------|------|
| `register_cpu_ci` | CPU-only 单元测试，无需 GPU |
| `register_cuda_ci` | NVIDIA GPU 测试（绝大多数测试用此函数） |
| `register_amd_ci` | AMD ROCm GPU 测试（仅 AMD 特定代码路径） |
| `register_npu_ci` | 华为 Ascend NPU 测试（仅 NPU 特定代码路径） |
| `register_xpu_ci` | Intel XPU 测试（仅 XPU 特定代码路径，不支持 `stage`/`runner_config`） |

> **suite 名称解析逻辑见 `python/sglang/test/ci/ci_register.py` 的 `effective_suite` 属性。** 有效 suite 速查表见 `reference/ci-suites.md`。

## 2. 测试结构

- [ ] 需要 GPU/Server 的测试继承 `CustomTestCase`；纯 CPU 单元测试（`test/registered/unit/`）允许使用 `unittest.TestCase`
- [ ] 包含 `if __name__ == "__main__": unittest.main()`
- [ ] 放在正确目录：
  - 不需要 server → `test/registered/unit/`
  - 需要 server → `test/registered/<category>/`
  - JIT 内核正确性 → `python/sglang/jit_kernel/tests/`
  - JIT 内核基准测试 → `python/sglang/jit_kernel/benchmark/`
- [ ] 除非有意为之，文件不在 `test/manual/` 下

## 3. Server 生命周期（仅限 Server 测试）

- [ ] `setUpClass` 通过 `popen_launch_server` 启动 server 或继承 `DefaultServerBase`
- [ ] `tearDownClass` 通过 `kill_process_tree` 终止 server
- [ ] `tearDownClass` 是**防御性的**：访问 `cls.process` 等资源前使用 `hasattr` / null 检查（`setUpClass` 可能未完成分配）
- [ ] Server 启动超时合理（默认 `DEFAULT_TIMEOUT_FOR_SERVER_LAUNCH` = 600s）
- [ ] 端口冲突已避免：使用 `DEFAULT_URL_FOR_TEST`（自动分配端口）

```python
# from sglang.test.test_utils import CustomTestCase, popen_launch_server
# from sglang.srt.utils import kill_process_tree  # 定义在 utils/common.py

# 正确的 setUpClass 模式（防御性初始化）：
@classmethod
def setUpClass(cls):
    try:
        cls.process = popen_launch_server(...)
    except Exception:
        cls.process = None
        raise  # 保留异常以便 CI 报错

# 正确的 tearDownClass 模式（防御性清理）：
@classmethod
def tearDownClass(cls):
    if hasattr(cls, "process") and cls.process:
        kill_process_tree(cls.process.pid)
```

## 4. 模型选择

- [ ] 使用满足测试需求的**最小模型**：
  - 模型无关特性 → `DEFAULT_SMALL_MODEL_NAME_FOR_TEST`（1B）
  - Base 模型测试 → `DEFAULT_SMALL_MODEL_NAME_FOR_TEST_BASE`
  - 性能测试 → `DEFAULT_MODEL_NAME_FOR_TEST`（8B）
  - MoE 专项 → `DEFAULT_MOE_MODEL_NAME_FOR_TEST`
  - 视觉语言 → `DEFAULT_SMALL_VLM_MODEL_NAME_FOR_TEST`
- [ ] 不是"以防万一"使用更大的模型 — 浪费 CI 时间

## 5. 测试质量

### 需要标记的反模式
- [ ] 测试验证 Python 本身是否工作（如"dataclass 存储字段？"）→ 删除
- [ ] 测试只验证 mock 工作（如"mock 返回我设置的值"）→ 测试真实逻辑
- [ ] 不管正确与否始终通过的测试 → 检查断言
- [ ] 使用 `time.sleep()` 等待异步操作 → 使用正确的同步机制
- [ ] 硬编码端口 → 使用 `DEFAULT_URL_FOR_TEST`
- [ ] 硬编码文件路径 → 使用临时目录或 `__file__` 相对路径

### 覆盖率缺口检查
- [ ] 新的公共函数/类 → 至少有一个单元测试
- [ ] 新的 API 端点 → 在 `test/registered/openai_server/` 有 E2E 测试
- [ ] 新的内核 → 有正确性测试 + 基准测试
- [ ] 新的 CLI 参数 → 有参数校验测试
- [ ] Bug 修复 → 有能捕获原始 bug 的回归测试

### 断言质量
- [ ] 断言具体：`assertEqual`、`assertGreater` — 不只是 `assertTrue`
- [ ] 断言信息有描述性：`self.assertEqual(x, 5, "Layer norm 输出不匹配")`
- [ ] 无注释掉的断言（要么启用要么删除）
- [ ] 浮点比较使用适当容差（`assertAlmostEqual`、`torch.allclose`）

## 6. 单元测试（无 Server）

- [ ] 使用 `unittest.mock.patch` / `MagicMock` 替代外部依赖
- [ ] 不导入仅限 GPU 的包（除非做了 stub，否则测试必须在 CPU CI 上可运行）
- [ ] 如需 stub GPU 包：使用 `patch.dict("sys.modules", ...)` 作为类装饰器或用 `start`/`stop`，不在模块级直接修改 `sys.modules`
- [ ] 放在 `test/registered/unit/` 下，镜像 `python/sglang/srt/` 的结构
- [ ] 优先使用 `register_cpu_ci` 而非 `register_cuda_ci`

## 7. E2E 测试（需要 Server）

- [ ] 优先使用 `DefaultServerBase` 或专用 fixture，而非手动 `setUpClass`（当前代码库 fixture 使用率较低，此为渐进方向，不阻塞提交）
- [ ] Server 参数最小化：只传特性相关参数，不是完整生产配置
- [ ] 测试真实 HTTP 行为（状态码、响应 schema），而非仅"不崩溃"
- [ ] 超时和重试逻辑：测试失败时不会永远挂起
- [ ] 资源清理：临时文件、额外进程等

## 8. 基准测试

- [ ] 有合理的性能阈值（不只是"不报错就行"）
- [ ] 使用真实的输入大小和 batch 维度
- [ ] 测量前包含预热迭代
- [ ] 注册到适当的基准测试 suite
- [ ] 以标准格式报告结果，便于 CI 追踪

## 9. 精度评估测试

- [ ] 使用适当的评估 mixin（`MMLUMixin`、`HumanEvalMixin`、`GSM8KMixin` 等）
- [ ] 分数阈值现实且已文档化
- [ ] `num_examples` 合理（CI 预算：64-256 个样本，不是完整数据集）
- [ ] 注册到适当 suite（H100/large 用于 8B 模型）

## 10. Model Gateway 测试（`sgl-model-gateway/tests/`、`sgl-model-gateway/e2e_test/`）

### Rust 单元/集成测试
- [ ] 测试使用 `#[tokio::test]` 或 `#[test]` 标注，异步测试使用正确的 runtime
- [ ] Mock 依赖：使用 `tower` Service builder 或自定义 mock server（本项目不使用 `mockito`），不依赖真实上游
- [ ] 断言具体：使用 `assert_eq!`/`assert!(expr).context("msg")`，不裸 `assert!`
- [ ] 测试覆盖：路由策略、负载均衡、故障转移、限流、认证均有测试
- [ ] 并发测试使用 `serial_test` crate 控制执行顺序，避免端口冲突

### E2E 测试（`sgl-model-gateway/e2e_test/`）
- [ ] 测试启动真实的 gateway 实例和 mock backend
- [ ] 验证 HTTP 状态码、响应 schema、流式行为
- [ ] 测试后有清理（进程、端口、临时文件）
- [ ] 不硬编码端口或地址

## 11. JIT 内核基准测试（`python/sglang/jit_kernel/benchmark/`）

- [ ] 使用满足测试需求的**最小模型**（遵循第 4 节模型选择规则）
- [ ] benchmark 脚本有合理的默认参数和 `--help` 文档
- [ ] 结果输出格式标准化，可被 CI 或比较工具解析

> **严重等级定义及映射见 SKILL.md 的"严重等级定义"节和"跨维度严重等级映射"节。**
