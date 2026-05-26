# 命名规范参考

检视 staged 变更中的命名时交叉引用本文件。每节对应一个特定子系统的命名约定。

> **最后验证：** 2026-05（与 `arg_groups/` 实际目录结构同步校验）

---

## 1. 推测解码（Speculative Decoding）

**适用范围：** `python/sglang/srt/speculative/`、相关注意力后端、调度器累加器、IPC 字段、可观测性指标、CLI 标志。

**来源：** `.claude/rules/speculative-naming.md`（最后同步：2026-05）

### 规则 1 — 动词形式，去掉 `-ed`
使用 `accept`，不用 `accepted`。

| 不要 | 要 |
|---|---|
| `num_accepted_tokens` | `num_accept_tokens` |
| `accepted_indices` | `accept_indices` |
| `accepted_token_ids` | `accept_tokens` |

### 规则 2 — 额外 token 命名为 `bonus_token` / `bonus_tokens`
目标模型额外生成的"+1" token 是**bonus token**。

| 不要 | 要 |
|---|---|
| `verified_id` / `verified_ids` | `bonus_token` / `bonus_tokens` |
| `output_id`（指 bonus 时） | `bonus_token` / `bonus_tokens` |

### 规则 3 — `accept` 包含 bonus；`correct` 不包含 bonus

| 动词 | 含义 |
|---|---|
| `accept_*` | 包含 bonus token |
| `correct_*` | 仅 draft，不含 bonus |

首选默认组合：`accept_tokens`、`correct_drafts`。

### 规则 4 — `num_` 表示计数；`_ct` 表示计数器；`_rate` 表示比率

| 形式 | 模式 | 含义 | 示例 |
|---|---|---|---|
| 计数 | `num_X` | 某时刻的快照量 | `num_accept_tokens` |
| 计数器 | `X_ct` | 单调递增的累加器 | `spec_verify_ct` |
| 比率 | `X_rate` | `[0, 1]` 范围的分数 | `accept_rate` |
| 数据 | 无前缀 | 实际数据 | `accept_tokens` |

不可混用：无 `num_X_ct`、无 `num_accept_rate`。

### 规则 5 — 在推测解码范围内去掉 `_token_id` / `_token_ids`
推测解码中 `_token_id` 是冗余的。

| 不要 | 要 |
|---|---|
| `accepted_token_ids` | `accept_tokens` |
| `curr_token_id` | `current_token` |
| `out_token_ids` | `out_tokens` |

### 规则 6 — 单数与复数
非标量 tensor 用复数；标量（kernel 内 `tl.load` 结果等）才用单数。

### 例外（保持不变）
- `accept_rate` / `accept_length` — 遵循论文约定
- `req.output_ids` — 完整请求输出历史，与推测解码无关
- 框架级命名：`image_token_id`、`pad_token_id`、`eos_token_id` 等

---

## 2. 模型注册

**适用范围：** `python/sglang/srt/models/`

### 注册命名
- 模型类名应匹配模型架构（如 `Llama3ForCausalLM`、`DeepseekV2ForCausalLM`）
- 注册条目使用 HuggingFace 模型 ID 模式匹配
- 自回归模型用 `ForCausalLM` 后缀，多模态模型用 `ForConditionalGeneration`

### 权重映射
- 使用 `WeightLoader` 显式映射 HF 权重名到 SGLang 参数名
- 映射键遵循 HF 命名约定（如 `model.layers.0.self_attn.q_proj.weight`）
- SGLang 内部名使用下划线命名（如 `qkv_proj_weight`）

---

## 3. Server 参数

**适用范围：** `python/sglang/srt/server_args.py`

### CLI 标志命名
- CLI 标志使用 `--kebab-case`
- Python 属性名使用 `snake_case`
- 布尔标志：`--enable-foo` / `--disable-foo`（不是 `--foo`/`--no-foo`）
- 禁用标志：`--disable-cuda-graph`

### 参数辅助模块（`arg_groups/`）

`python/sglang/srt/arg_groups/` 存放 argparse 动作和参数校验 hook，**不是** dataclass 分组。所有参数仍定义在 `server_args.py` 的 `ServerArgs` 中。

- **Argparse Action**（`argparse_actions.py`）：自定义 `argparse.Action` 子类，如 `LoRAPathAction`、`DeprecatedAction`、`DeprecatedStoreTrueAction`、`DeprecatedAliasStoreAction`
- **参数校验 Hook**（`*_hook.py`）：在 CLI 解析后执行校验/转换逻辑，如 `speculative_hook.py` 等（以实际目录为准）
- 文件名使用 `snake_case.py`，hook 文件以 `_hook.py` 后缀
- Hook 函数命名遵循 `<verb>_<feature>` 模式，常见动词：`apply_`（设置默认值）、`validate_`（校验约束）、`handle_`（综合处理）。示例：`apply_deepseek_v4_defaults`、`validate_hisparse`、`handle_speculative_decoding`
- Argparse Action 类名使用 `PascalCase` + `Action` 后缀（如 `LoRAPathAction`、`DeprecatedAction`）

---

## 4. 内核命名

**适用范围：** `python/sglang/jit_kernel/`

> **注意：** `sgl-kernel/` 不在 NPU 适配的检视范围内，AOT 内核命名规则仅供上游同步参考。

### JIT 内核（Triton）函数命名
- 使用 `动词_名词` 模式：`fuse_residual`、`rmsnorm_forward`
- kernel 函数加下划线前缀：`_rmsnorm_kernel`
- 包装函数（公开 API）无前缀：`rmsnorm_forward`

### 文件命名
- 内核文件：Triton 用 `snake_case.py`，CUDA 用 `snake_case.cu`
- 测试文件：`test_*.py`
- 基准测试文件：`bench_*.py`

---

## 5. 测试命名

**适用范围：** `test/registered/`、`python/sglang/jit_kernel/tests/`

### 文件命名
- 测试文件：`test_<特性>.py` 或 `test_<类别>_<特性>.py`
- 必须以 `test_` 开头以支持自动发现

### 类命名
- `Test<特性>` 或 `Test<特性><变体>`
- 示例：`TestSamplingPenalty`、`TestEagleSpecDecoding`

### 方法命名
- `test_<行为>` — 描述性的，不只是 `test_1`、`test_2`
- 示例：`test_repeat_penalty_positive`、`test_empty_input_raises`

### CI 注册命名
- Suite 名称格式：`base-{a,b,c}-test-{gpu数量}-gpu-{硬件}`（如 `base-b-test-1-gpu-large`）
- 新风格：使用 `stage` + `runner_config` 两参数，由 `ci_register.py` 拼接为 `{stage}-test-{runner_config}`
- AMD suite：`stage-{a,b,c}-test-{gpu数量}-gpu-small-amd`（如 `stage-b-test-1-gpu-small-amd`）
- NPU suite：`per-commit-{npu数量}-npu-{型号}`，型号为 `a2`（Ascend 910B）或 `a3`（Ascend 910C）（如 `per-commit-1-npu-a2`、`per-commit-4-npu-a3`）

---

## 6. 通用 Python 命名

**适用范围：** 仓库中所有 Python 代码。

- 函数、方法、变量、模块名使用 `snake_case`
- 类名使用 `PascalCase`
- 常量使用 `UPPER_SNAKE_CASE`
- 私有成员：`_前缀`
- 名称改写（name mangling）：`__双前缀`（少用，尽量避免）
- 魔术方法：`__init__`、`__repr__` 等（仅用于标准协议）

### 导入约定
- `from sglang.srt.xxx import Yyy` — 优先使用绝对导入
- 避免相对导入（`from .xxx import Yyy`），紧密耦合的包除外
- `isort` 配置 `profile=black` 和 `known_first_party=sglang` — 不要对抗排序器
