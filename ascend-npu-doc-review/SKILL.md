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
| 变更文件中含动态字段占位符（正文或代码块均算） | M-F-2 |
| 变更文件中含链接 | M-F-5 |
| 删除 .mdx 文件 | M-F-6 |
| 新增 .mdx 文件 | M-F-7 |
| 全部 .mdx 变更 | A-1, A-2 |

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
| **重要** | [references/rules-mandatory.md](references/rules-mandatory.md) | 9 条，必须全部通过才可合入 | M-C-1 正确性 → 🔴 BLOCK；M-F-1~M-F-8 重要格式 → 🟡 ISSUE |
| **建议** | [references/rules-advisory.md](references/rules-advisory.md) | 2 条（A-1 ~ A-2），尽量满足 | 💡 SUGGESTION |

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

## 核心原则

- **重要规则必须全部通过**，不通过即阻止合入
- **参考最佳实践**：检查规则中涉及"参考最佳实践"时，以 `best_practice/deepseek_r1.mdx` 为基准模板
- **占位符统一**：动态字段的占位符格式应尽量与最佳实践中已有格式保持一致
- **脚本注释不可缺**：凡涉及脚本，必须检查开头注释块中是否完整说明需修改的动态字段
- **链接须可解析**：站内链接（root-relative `/docs/...`）、跨文件锚点、同文件锚点均须确认目标存在；旧站 `docs.sglang.io` 绝对链接应改写为 root-relative
- **仅审查 ascend-npus 目录下文件**：非该路径的文件不纳入检查，但 `docs.json` 始终纳入（用于 M-F-6 / M-F-7）
