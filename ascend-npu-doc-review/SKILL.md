---
name: ascend-npu-doc-review
description: 检视 SGLang Ascend NPU 文档 PR 的规范性。涵盖脚本占位符、动态字段、硬编码 IP/路径、链接有效性、docs.json 同步、环境变量格式等检查。当用户提供 sgl-project/sglang 仓库中涉及 /docs_new/docs/hardware-platforms/ascend-npus/ 路径的 PR 号或 PR 链接要求审查时触发。
---

# Ascend NPU 文档 PR 检视

对 `sgl-project/sglang` 仓库中 `docs_new/docs/hardware-platforms/ascend-npus/` 路径下的文档 PR 进行规范性审查。

## Usage

当用户提供 PR 号或 PR URL 并要求审查 Ascend NPU 文档时触发（基于本 skill 的 description 自动匹配，无需注册斜杠命令）。常见触发表述：

```
/ascend-npu-doc-review <PR 号>
/ascend-npu-doc-review <PR URL>
审查这个 Ascend NPU 文档 PR：<PR 号 / PR URL>
```

## Steps

### 0. 解析 PR 标识

从 PR 号（如 `12345`）或 PR URL（如 `https://github.com/sgl-project/sglang/pull/12345`）中提取 PR 号 `<N>`。

### 1. 获取 PR 变更数据

**方式一（优先）：`gh` CLI**

```bash
gh pr view <N> --repo sgl-project/sglang --json title,body,files,author,baseRefName,headRefName
gh pr diff <N> --repo sgl-project/sglang
```

- `--json files` 提供结构化文件列表（含 `path` / `status` / `additions` / `deletions`），可直接解析变更类型（added / modified / removed / renamed）
- `gh pr diff` 提供完整 unified diff，用于逐行审查
- 可额外获取 PR 元数据：`title`、`author`、`reviews` 等

**方式二（降级）：raw diff + API（`gh` 不可用时）**

无需安装任何工具，通过 WebFetch 直接访问 GitHub 公开端点：

```
WebFetch: https://github.com/sgl-project/sglang/pull/<N>.diff
WebFetch: https://api.github.com/repos/sgl-project/sglang/pulls/<N>/files
```

- `.diff` 端点返回完整 unified diff
- `/files` API 返回结构化 JSON（字段：`filename`、`status`、`additions`、`deletions`），无需认证即可调用，频率限制为 60 req/h（对 PR 审查完全够用）
- 从 diff 的 `diff --git a/<path> b/<path>` 行也可解析变更文件列表和状态（new file = added，deleted file = removed，index = modified）

### 2. 过滤变更范围

从文件列表中筛选 `docs_new/docs/hardware-platforms/ascend-npus/` 路径下的 `.mdx` 文件，以及 `docs_new/docs.json`。

- 如果筛选后零匹配，报告 "该 PR 不涉及 Ascend NPU 文档变更" 并退出
- `docs.json` 始终纳入审查范围（用于 M-6 / M-7），即使不在 ascend-npus 子目录下
- 非 ascend-npus 路径的其余变更文件不纳入审查，但可作为背景信息参考

### 3. 逐规则检查

阅读 [references/rules-mandatory.md](references/rules-mandatory.md) 和 [references/rules-advisory.md](references/rules-advisory.md)，对每个变更文件逐规则检查。

**规则适用性自动判断**：

| 触发条件 | 适用规则 |
|----------|----------|
| 变更文件中含脚本代码块（```bash / ```shell / ```python） | M-C-1, M-F-1, M-F-3, M-F-4, M-F-8 |
| 变更文件在 `best_practice/` 或 `model-tutorials/` 下且含脚本 | M-F-9, A-4 |
| 变更文件在 `best_practice/` 下 | M-F-10 |
| 变更文件中含 `sglang.launch_server` / `sglang serve` 命令 | A-3 |
| 变更文件中含动态字段占位符（正文或代码块均算） | M-F-2 |
| 变更文件中含链接 | M-F-5 |
| 删除 .mdx 文件 | M-F-6 |
| 新增 .mdx 文件 | M-F-7, M-F-11 |
| 全部 .mdx 变更 | A-1, A-2, A-5, M-F-12 |

> **推荐检查顺序**：先跑正确性规则（M-C，可能直接阻断），再跑重要格式规则（M-F），最后跑建议规则（A）。

### 4. 输出报告

按文件汇总问题，给出整体判定。

## 检视流程

### 阶段一：范围确认

1. 从 PR diff 获取变更清单
2. 区分新增、修改、删除三类操作
3. 确认 `docs.json` 是否在变更范围内（M-F-6 / M-F-7 需要）

### 阶段二：逐规则检查

按重要级别优先，逐文件对照规则审查：

| 优先级 | 规则文件 | 说明 | 严重级别 |
|--------|----------|------|----------|
| **重要** | [references/rules-mandatory.md](references/rules-mandatory.md) | 13 条，必须全部通过才可合入 | M-C-1 正确性 → 🔴 BLOCK；M-F-1~M-F-12 重要格式 → 🟡 ISSUE |
| **建议** | [references/rules-advisory.md](references/rules-advisory.md) | 5 条（A-1 ~ A-5），尽量满足 | 💡 SUGGESTION |

> **严重级别映射**（输出报告统一使用此三档）：
> - 🔴 **BLOCK** — 正确性规则（M-C-x）。用户按文档操作会直接报错，必须修复。
> - 🟡 **ISSUE** — 重要格式规则（M-F-x）。脚本能跑通但影响规范性/可维护性，合入前必须修复。
> - 💡 **SUGGESTION** — 建议规则（A-x）。尽量满足，不阻止合入。

**特别关注**：当变更涉及脚本代码块时，M-C-1、M-F-1 ~ M-F-4、M-F-8 需逐条覆盖。当变更涉及链接时，M-F-5 必查。当变更涉及新增/删除 `.mdx` 文件时，M-F-6 / M-F-7 必查。

### 阶段三：输出审查意见

**必须全英文输出**，可直接粘贴到 GitHub PR comment 中。

**输出结构**：按文件分组，每文件输出：

1. **文件路径**
2. **一个整体的 comment block**，内容为：

````markdown
**Severity:** 🔴 BLOCK / 🟡 ISSUE / 💡 SUGGESTION

**Issue:** <具体问题描述>

**Fix:** <可操作的修复指引，含 diff 或代码片段>
````

每条规则违反对应一个独立的 comment block，同一文件下有多个问题时连续排列。

**完整输出示例**：

---
**File:** `docs_new/docs/hardware-platforms/ascend-npus/best_practice/qwen3_30b_a3b.mdx:74`

````markdown
**Severity:** 🔴 BLOCK

**Issue:** `export HCCL_ALGO=level0:NA;level1:ring` — the semicolons in the value are not quoted. The shell will interpret `level1:ring` as a separate command, causing a runtime error.

**Fix:** Wrap the value in double quotes:
`export HCCL_ALGO="level0:NA;level1:ring"`
````

---

**File:** `docs_new/docs/hardware-platforms/ascend-npus/best_practice/qwen3_30b_a3b.mdx:45-90`

````markdown
**Severity:** 🟡 ISSUE

**Issue:** The script block starting at line 45 is missing the dynamic field comment block. Users won't know which variables to replace before running.

**Fix:** Add the following block at the top of the script:
```bash
# ============================================================
# Before running, update the following variables:
#   MODEL_PATH: path to the model weights directory
#   HCCL_SOCKET_IFNAME: network interface name for HCCL
# ============================================================
```
````

---

**整体判定**：**APPROVE** / **REQUEST CHANGES** / **BLOCKED**

- APPROVE：无 🔴 BLOCK 且无 🟡 ISSUE（仅余 💡 SUGGESTION 或全部通过）
- REQUEST CHANGES：无 🔴 BLOCK，但存在一个或多个 🟡 ISSUE（M-F 重要格式问题）
- BLOCKED：存在 🔴 BLOCK（M-C 正确性问题，用户照做会直接报错）

## 并行执行策略

当 PR 涉及多个变更文件时，使用 subagent 并行化审查以缩短耗时。

### 任务拆分

审查规则按依赖关系分为两类：

| 类别 | 规则 | 特点 | 并行方式 |
|---|---|---|---|
| **逐文件规则** | M-C-1, M-F-1~M-F-4, M-F-8~M-F-10, A-1~A-4 | 仅需单个文件内容即可判断，文件间无依赖 | 每文件一个 subagent，全部并行派发 |
| **跨文件规则** | M-F-5, M-F-6, M-F-7 | 需跨文件 Glob/Grep 或对比 docs.json | 各一个 subagent，与逐文件 subagent 并行派发 |

### 执行步骤

#### 1. 准备阶段（主控，串行）

- 获取 PR diff 和文件列表（Step 0-2）
- 按 [规则适用性自动判断](#3-逐规则检查) 表确定每个变更文件触发哪些规则
- 准备好每个 subagent 的输入（文件路径 + 触发规则列表 + diff 内容）

#### 2. 并行执行（一次性派发所有 subagent）

并行启动以下 subagent，互不依赖，同时运行：

| subagent 名称 | 类型 | 输入 | 负责规则 |
|---|---|---|---|
| `per-file-<file-slug>` | 每文件 × N | 单个 `.mdx` 文件路径 + diff + 适用规则列表 | 该文件触发的所有逐文件规则 |
| `cross-links` | 跨文件 × 1 | 全部变更文件列表 + ascend-npus 目录路径 | M-F-5 |
| `cross-docs-json` | 跨文件 × 1 | docs.json diff + 所有 .mdx 变更清单 | M-F-6, M-F-7 |

**per-file subagent 提示词模板**：

```
You are reviewing a single file in an Ascend NPU docs PR for sgl-project/sglang.
Apply ONLY the rules listed below to the provided file diff. Report violations.

Rules to apply: <RULE_ID_LIST>

--- Rule Summaries ---
(Each rule's key check and severity, extracted from rules-mandatory.md / rules-advisory.md)

--- File Info ---
Path: <FILE_PATH>
Status: added|modified|deleted

--- PR Diff (this file only) ---
<FILE_DIFF_CONTENT>

For each violation, output exactly:
- Rule: <ID>
- Severity: 🔴 BLOCK / 🟡 ISSUE / 💡 SUGGESTION
- Location: line <N> or lines <N>-<M>
- Issue: <one-line description>
- Fix: <actionable suggestion>
```

**cross-links subagent 提示词模板**：

```
Verify all links in the changed files under docs_new/docs/hardware-platforms/ascend-npus/.
Classify each link as: internal root-relative (/docs/...), internal relative, same-file anchor, external, old docs.sglang.io.
Tasks:
1. For internal links: verify the target file/anchor exists by glob/grep in ascend-npus/ directory
2. Flag old docs.sglang.io links: suggest rewriting to root-relative /docs/...
3. Check same-file anchors resolve to matching heading/id
Report violations with file path, line number, link text, link URL, and suggested fix.
```

**cross-docs-json subagent 提示词模板**：

```
Check docs.json synchronization in this PR.
Tasks:
1. For each DELETED .mdx under ascend-npus/: verify the corresponding navigation entry and redirects entry are removed from docs.json
2. For each ADDED .mdx under ascend-npus/: verify a pages entry exists in the correct group (group: "Ascend NPUs", sub-groups: "Model Tutorials" / "Best Practice")
3. For modified files: no docs.json check needed
Report violations with file path, missing/extra entry details, and suggested fix.
```

#### 3. 结果汇总（主控，串行）

- 等待所有 subagent 返回
- 按文件分组合并问题列表
- 按严重级别排序（🔴 BLOCK → 🟡 ISSUE → 💡 SUGGESTION）
- 按 [阶段三](#阶段三输出审查意见) 格式输出最终报告和整体判定

### 调度原则

- **无依赖即并行**：逐文件规则和跨文件规则之间无依赖，全部同时派发
- **M-F-10 不拆分**：用例一致性检查涉及单个文件内的多段比对（标题、元数据、表格、bench 命令），由一个 per-file subagent 整体完成，不二次拆分
- **docs.json 变更即触发**：即使只有 docs.json 变更而无 .mdx 变更，也要派发 cross-docs-json subagent 检查 M-F-6 / M-F-7
- **单文件 PR 同样适用**：即使只有 1 个变更文件，仍用并行框架（1 个 per-file + cross-links + cross-docs-json），流程统一

## 核心原则

- **重要规则必须全部通过**，不通过即阻止合入
- **参考最佳实践**：检查规则中涉及"参考最佳实践"时，以 `best_practice/deepseek_r1.mdx` 为基准模板
- **占位符统一**：动态字段的占位符格式应尽量与最佳实践中已有格式保持一致
- **脚本注释不可缺**：凡涉及脚本，必须检查开头注释块中是否完整说明需修改的动态字段
- **链接须可解析**：站内链接（root-relative `/docs/...`）、跨文件锚点、同文件锚点均须确认目标存在；旧站 `docs.sglang.io` 绝对链接应改写为 root-relative
- **仅审查 ascend-npus 目录下文件**：非该路径的文件不纳入检查，但 `docs.json` 始终纳入（用于 M-F-6 / M-F-7）
