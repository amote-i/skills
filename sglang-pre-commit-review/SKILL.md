---
name: sglang-pre-commit-review
description: SGLang 仓库提交前检视（Ascend NPU 适配 fork）。从代码、测试、文档、构建系统四个维度检查待提交变更，输出按严重等级分类的检视报告。使用中文触发词（如 /检视、帮我看看改动）时自动激活。
---

# Skill: sglang-pre-commit-review

在 SGLang Ascend NPU 适配仓库中对变更进行检视。从**代码、测试、文档、构建系统**四个维度分别分析，输出按严重等级分类的检视报告。

## 触发条件

当用户说以下内容时使用本 skill：
- `/review`、`/检视`、`review staged`、`提交前检视`、`review 一下`、`帮我看看改动`、`帮我 review`、`看看这些改动`
- 用户要求在提交前或推送前检查变更内容
- 用户希望在提交 PR 前做最后一轮检查
- 用户指定了 commit 范围（如 `/检视 HEAD~3..HEAD`、`/review abc123..def456`）

## 核心流程

```
0. 解析   — 检查用户是否指定了 commit 范围（如有则跳过自动探测）
1. 探测   — 判断变更来源（未提交 → 未推送 → 无变更）
2. 作用域 — 仅保留 docs_new/、python/、sgl-model-gateway/、test/、docker/ 内的文件
3. 过滤   — 排除二进制、生成文件、大型 diff、第三方代码
4. 分类   — 按维度对变更文件归类（代码、测试、文档、构建系统）
5. 检视   — 按维度加载对应检查清单并逐项检查
6. 报告   — 输出按严重等级分类的检视报告
```

### 第一步：探测变更来源

按优先级依次检查，命中第一个有内容的即停止：

```bash
# 优先级 0：检查用户是否指定了 commit 范围
# 如果用户消息中包含 ".." 格式的 commit range，直接进入情况 D

# 优先级 1：检查未提交的变更（staged + unstaged）
git status --porcelain
git diff --staged --stat
git diff --stat

# 优先级 2：检查未推送的提交（先检查 upstream 是否存在）
git rev-parse --abbrev-ref @{upstream} 2>/dev/null
# 如果 upstream 存在：
git log @{upstream}..HEAD --oneline
git diff @{upstream}..HEAD --stat

# 优先级 3：upstream 不存在时，检查本地所有分支上的独有提交
# 注意：此命令会扫描所有本地分支，在有多条陈旧本地分支时可能产生大量输出
git log --branches --not --remotes --oneline
```

#### 情况 A：存在未提交的变更（staged + unstaged）

**说明：** 检测到未提交的变更，优先检视这些内容。未推送的提交暂不纳入本轮检视范围。

采集方式 — 同时覆盖 staged 和 unstaged：
```bash
# 变更概况（HEAD 为基准，涵盖 staged + unstaged）
git diff HEAD --stat
git diff HEAD --name-only

# 完整 diff（用于详细检视）
git diff HEAD
```

#### 情况 B：无未提交变更，但存在未推送的提交

**说明：** 工作区和暂存区干净，但本地有尚未推送到远程的提交。检视范围覆盖自 `@{upstream}` 以来的所有本地提交。

如果分支有 upstream：
```bash
# 涉及的提交列表
git log @{upstream}..HEAD --oneline --reverse

# 完整 diff
git diff @{upstream}..HEAD --stat
git diff @{upstream}..HEAD
git diff @{upstream}..HEAD --name-only
```

如果分支无 upstream（新建分支、detached HEAD）：
```bash
# 动态探测远程默认分支名（main / master / 其他），优先使用本地 symref 避免网络调用
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
if [ -z "$DEFAULT_BRANCH" ]; then
    DEFAULT_BRANCH=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | sed 's/.*: //')
fi
BASE_REF="${DEFAULT_BRANCH:-main}"

# 使用所有本地独有提交
git log --branches --not --remotes --oneline --reverse
git diff $(git merge-base "origin/${BASE_REF}" HEAD 2>/dev/null || git rev-list --max-parents=0 HEAD)..HEAD --stat
git diff $(git merge-base "origin/${BASE_REF}" HEAD 2>/dev/null || git rev-list --max-parents=0 HEAD)..HEAD --name-only
git diff $(git merge-base "origin/${BASE_REF}" HEAD 2>/dev/null || git rev-list --max-parents=0 HEAD)..HEAD
```

#### 情况 C：本地分支与远程一致

**说明：** 当前分支与远程追踪分支完全同步，没有未提交的变更，也没有未推送的提交。无需检视。

输出并停止：
```
当前分支与远程一致，没有未提交或未推送的变更，无需检视。
如需检视特定提交范围，请指定 commit 范围（如 /检视 HEAD~3..HEAD）。
```

#### 情况 D：用户指定了 commit 范围

**说明：** 用户在消息中指定了明确的 commit 范围（如 `HEAD~3..HEAD`、`abc123..def456`）。直接检视该范围内的变更，不做自动探测。

采集方式（以 `<range>` 表示用户指定的范围）：
```bash
# 涉及的提交列表
git log <range> --oneline --reverse

# 完整 diff
git diff <range> --stat
git diff <range> --name-only
git diff <range>
```

> **注意：** `git diff <range>`（两点语法）显示两端点之间的累积差异，已包含 merge 引入的变更。若需按每个 merge commit 分别查看其引入的变更，可对每个 merge commit 执行 `git diff <merge-sha>^1 <merge-sha>`（对比 merge 与其第一父 commit）。

### 检视作用域（NPU 适配分支）

> **本仓库为 Ascend NPU 适配 fork，仅检视以下 5 个目录内的变更：**
>
> | 允许检视的目录 | 对应检查维度 |
> |---------------|-------------|
> | `docs_new/` | 文档 |
> | `python/` | 代码（srt、jit_kernel、lang、cli、arg_groups 等全部子目录） |
> | `sgl-model-gateway/` | 代码 + 测试 |
> | `test/` | 测试 |
> | `docker/` | 构建系统 |
>
> **不在上述目录中的文件一律跳过**，包括但不限于：`sgl-kernel/`、`rust/`、`proto/`、`benchmark/`、`.github/`、`scripts/`、`examples/`、`assets/`、`3rdparty/`、顶层配置文件等。这些目录由上游维护或 NPU 适配暂不涉及。
>
> 在过滤阶段，先将所有不在作用域内的文件标记为"作用域外"并跳过，然后再应用下方通用过滤规则。

### 变更过滤

在执行详细检视前，过滤掉以下不需要检视的文件：

| 跳过类型 | 判断方式 |
|----------|----------|
| 作用域外文件 | 不在 `docs_new/`、`python/`、`sgl-model-gateway/`、`test/`、`docker/` 下的文件 |
| 二进制文件 | 无 text diff 输出（图片、模型权重、`.bin`、`.safetensors`、`.so`、`.dll`、`.o`） |
| 自动生成文件 | `*.pyc`、`*.pyo`、`*.pyd`、`__pycache__/`、`*.egg-info/`、`*.egg`、`*.whl`、`poetry.lock`、`package-lock.json`、`.DS_Store` |
| 大型 diff | 单文件 diff 超过 2000 行时：若纯为移动/重命名，跳过检视并标注；若有逻辑变更，仅检视文件头部（import、类定义、函数签名）及实际变更行，报告时标注"文件过大，部分检视" |
| 第三方代码 | `third_party/`、`3rdparty/`、`venv/`、`node_modules/` |

过滤后在报告中标注：
```
**已跳过：** X 个作用域外文件（列出关键文件名）、Y 个二进制文件、Z 个生成文件、W 个大型 diff（部分检视）、V 个第三方文件（列出文件名）
```

### Diff 采集策略

> **重要：** 完整 diff（如 `git diff HEAD`）可能在变更文件较多时超出工具输出限制导致截断。**必须采用两步采集法**：

1. **第一步：概览** — 先执行 `git diff <base> --stat` 和 `git diff <base> --name-only` 获取变更文件列表和行数概况
2. **第二步：逐文件详细获取** — 对每个需要检视的文件，分别执行 `git diff <base> -- <file>` 获取完整 diff，避免单次输出截断

此策略适用于上述所有情况（A/B/D）的详细 diff 采集。

### 第二步：变更文件分类

将每个变更文件归入一个或多个维度：

| 维度 | 文件路径模式 |
|------|-------------|
| **代码** | `python/sglang/srt/**/*.py`（含 `python/sglang/srt/arg_groups/**/*.py`）、`python/sglang/jit_kernel/**/*.py`、`sgl-model-gateway/src/**/*.rs`、`python/sglang/lang/**/*.py`、`python/sglang/cli/**/*.py` |
| **测试** | `test/**/*.py`（含 `test/registered/**/*.py`、`test/registered/unit/**/*.py`、`test/registered/ascend/**/*.py`、`test/registered/amd/**/*.py`、`test/srt/**/*.py`、`test/manual/**/*.py`）、`python/sglang/jit_kernel/tests/**/*.py`、`python/sglang/jit_kernel/benchmark/**/*.py`、`sgl-model-gateway/tests/**/*.rs`、`sgl-model-gateway/e2e_test/**/*.py` |
| **文档** | `docs_new/**/*.md`、`docs_new/**/*.mdx`、`docs_new/**/*.rst`、`docs_new/**/*.ipynb` |
| **构建系统** | `docker/**/*`、`python/pyproject.toml`、`python/setup.py`、`python/setup.cfg`、`sgl-model-gateway/Cargo.toml` |

### 第三步：按维度检视

对每个存在变更的维度，**使用 Read 工具读取对应的 checklist 文件**，然后按清单逐项检查 diff 中的变更：

- **代码变更** → 使用 Read 工具加载 `checklists/code.md`，按清单逐项检查
- **测试变更** → 使用 Read 工具加载 `checklists/tests.md`，按清单逐项检查
- **文档变更** → 使用 Read 工具加载 `checklists/docs.md`，按清单逐项检查
- **构建系统变更** → 使用 Read 工具加载 `checklists/build.md`，按清单逐项检查

> **重要：** checklist 文件路径相对于 skill 目录：`.claude/skills/sglang-pre-commit-review/checklists/`。命名规则参考文件在同目录的 `reference/naming-rules.md`。

### 第四步：输出报告

按以下格式输出结构化报告。

## 输出格式

```markdown
## 检视报告

**变更来源：** 未提交的变更 / 未推送的本地提交 / 用户指定范围
**变更文件数：** X 个文件（+Y/-Z 行）
**涉及维度：** 代码、测试、文档、构建系统（列出实际涉及的维度）
**已跳过：** X 个作用域外文件（列出关键文件名）、Y 个二进制文件、Z 个生成文件、W 个大型 diff（部分检视）、V 个第三方文件（如无则省略）

---

### [代码] 检视结果

#### 严重（必须修复）
- **[文件:行号]** 问题描述。影响说明。修复建议。

#### 重要（建议修复）
- **[文件:行号]** 问题描述。影响说明。修复建议。

#### 次要（可选改进）
- **[文件:行号]** 问题描述。改进建议。

---

### [测试] 检视结果

（同样按严重等级组织）

---

### [文档] 检视结果

（同样按严重等级组织）

---

### [构建系统] 检视结果

（同样按严重等级组织。如无构建系统变更则省略此节）

---

### 汇总

| 维度 | 严重 | 重要 | 次要 |
|------|------|------|------|
| 代码 |  X   |  X   |  X   |
| 测试 |  X   |  X   |  X   |
| 文档 |  X   |  X   |  X   |
| 构建系统 | X |  X   |  X   |

**结论：** 可以提交 / 需修复后提交 / 阻塞

**理由：**（1-2 句技术评估）
```

## 严重等级定义

| 等级 | 含义 | 示例 |
|------|------|------|
| **严重（Critical）** | 会导致故障、数据丢失或安全问题 | Bug、崩溃、结果错误、密钥泄露、API 契约破坏 |
| **重要（Important）** | 合并前应修复；质量/风险问题 | 缺少错误处理、新逻辑缺少测试、性能退化、功能不完整 |
| **次要（Minor）** | 可选改进；无功能影响 | 风格优化、小命名不一致、可选文档补充 |

## 检视规则

**应该做的：**
- 按实际严重等级分类 — 不要把所有问题都标为"严重"
- 引用具体 `文件:行号` 位置
- 解释每个问题**为什么**重要
- 肯定做得好的部分
- 交叉引用 SGLang 特有规则（命名、测试注册等）
- 检查缺失的关联变更（代码改了但测试/文档没更新）
- 给出明确结论

**不应做的：**
- 报告 pre-commit hooks 已覆盖的问题（格式化、import 排序、行尾空格）
- 把风格问题标为"严重"
- 对未审查的文件发表意见
- 含糊其辞（"改进错误处理"）— 要具体
- 跳过结论

## 跨维度检查

以下检查横跨多个维度，应全局应用：

1. **代码变更但测试未更新** → 重要："`file.py` 中的新逻辑没有测试覆盖。需在 `test/registered/<category>/` 添加测试。"
2. **代码变更但文档未更新** → 次要/重要（取决于范围）："新 CLI 参数 `--foo` 未在 `docs_new/` 中记录。"
3. **测试变更但无代码变更** → 确认测试是在修复误报还是在覆盖之前未测试的路径。
4. **新公共 API 缺少 docstring** → 次要：当前代码库 docstring 覆盖率较低（~22%），鼓励但不强制。仅对 `entrypoints/` 公共 API 层建议作为重要。
5. **新模型实现未注册** → 严重：模型将无法被发现。

### 跨维度严重等级映射

| 跨维度场景 | 严重 | 重要 | 次要 |
|------------|------|------|------|
| 代码→测试 | 新模型/新公共 API 零测试 | 新逻辑缺少测试覆盖 | 罕见边界情况未覆盖 |
| 代码→文档 | 新 CLI 参数无 help text | 新 API 端点未记录 | 新公共函数缺 docstring |
| 测试→代码 | — | 测试修复误报但代码仍有 bug | 测试覆盖已测路径 |
| 模型注册 | 模型未注册 | 注册信息不完整 | 命名不一致 |

## 与现有 Skill 的关系

本 skill 是补充而非替代：
- `write-sglang-test` — 提供编写测试的指导，本 skill 用于检视测试质量
- `requesting-code-review` — 用于请求外部人工 review
- `receiving-code-review` — 用于处理 reviewer 反馈
- `mechanical-refactor-verify` — 专门用于验证重构类 PR
