---
name: sglang-weekly-changelog
description: Use when the user asks to generate a weekly changelog by diffing two sglang commits to extract new server parameters and newly supported models, then writing results to the sglang-weekly repository.
---

# SGLang 每周变更日志

## 概述

通过 diff sglang 仓库的两个 commit，提取新增的 server 参数和新增支持的模型，将结果写入 sglang-weekly 仓库。

## 前提条件

- sglang 主线分支已是最新，且无未提交的修改。
- SSH key 已配置，可访问 `git@github.com:amote-i/sglang-weekly.git`。

## 工作流程

### 1. 准备 sglang-weekly 仓库

以下路径均相对于 sglang 项目根目录（即当前工作目录）。注意 `work_dirs/` 已在 sglang 的 `.gitignore` 中，其中的 sglang-weekly 是一个独立 git 仓库，不会污染主仓库。

1. 检查 `work_dirs/` 是否存在，不存在则创建。
2. 检查 `work_dirs/sglang-weekly/` 是否存在，不存在则执行 `git clone git@github.com:amote-i/sglang-weekly.git`。
3. 在 `work_dirs/sglang-weekly/` 中执行 `git checkout main && git pull` 确保内容最新。如果 pull 失败（如 detached HEAD），先执行 `git checkout main` 再 `git pull`。
4. 读取 `work_dirs/sglang-weekly/CHANGELOG.md`，最后一行包含 base commit id。
   - **首次运行边界情况：** 如果 `CHANGELOG.md` 不存在或为空（无历史记录行），则提示用户手动指定起始 commit，或使用 `git rev-list --max-parents=0 HEAD` 获取仓库首个 commit 作为 `last_commit`。

### 2. 确定 diff 范围

解析 CHANGELOG.md 最后一行，格式为 `YYYYMMDD.md: <完整commit-id>`，提取 last_commit。
在 sglang 仓库根目录执行 `git rev-parse HEAD` 获取 current_head（完整 40 位 commit SHA）。
diff 范围：`last_commit..current_head`。

### 3. 提取新增 server 参数

**目标文件：** `python/sglang/srt/server_args.py`

```bash
git diff <last_commit>..<current_head> -- python/sglang/srt/server_args.py
```

**在 diff 中查找的内容（仅关注新增 `+` 行中的 `parser.add_argument`）：**

| 字段 | 来源 | 说明 |
|---|---|---|
| 参数名称 | 第一个位置参数或 `name=`/`dest=` 中的 `--xx-yy` | 保留 `--xx-yy` 格式；如有多个名称则全部列出 |
| 类型 | `type=` 关键字 | `str`、`int`、`float`、`bool`；`action="store_true"` 表示 `Type: bool（set to enable）` |
| 默认值 | `default=` 关键字 | 可能引用 `ServerArgs.xxx`，需从类体中解析实际值 |
| 可选值 | `choices=` 关键字 | 可能引用变量，需解析变量值 |
| 描述 | `help=` 关键字 | 多行字符串需合并为一整句 |
| 分组/前一个参数 | `add_argument` 上方的注释行 `# Xxx options`，或前一个参数名称 | 有注释 → 新分组名称；无注释 → 前一个参数的 `--xx-yy` |

**默认值解析细节：**

- 当 `default=ServerArgs.xxx` 时，需在 `ServerArgs` 类体（文件顶部）中查找该属性的初始值。注意 `ServerArgs` 是 dataclass，值可能在 `__post_init__` 方法中被动态覆盖（如 `_handle_missing_default_values()` 补全 tokenizer 相关字段）。**若值在 `__post_init__` 中动态计算，输出时标注 `（set in __post_init__，初始值: xxx）`。**
- 当属性使用 `dataclasses.field(default_factory=...)` 时，默认值在运行时动态计算，无法静态解析。**输出时标注 `（computed at runtime: <factory 表达式>）`，并给出 factory 表达式内容作为参考。**
- 如果默认值引用了其他变量（如 `SAMPLING_BACKEND_CHOICES`），需解析该变量的实际值。

**可选值解析细节：** 当 `choices=[m.name.lower() for m in RealKvHashMode]` 时，需解析该枚举/类，列出实际的 choice 值。

### 4. 提取新增支持的模型

**目标文件（仅检查有新增内容的文件）：**

| 文件路径 | 章节标题 | 输出表格中的模型类型 |
|---|---|---|
| `docs_new/docs/supported-models/generative_models.mdx` | `## Supported models` | Large Language Model |
| `docs_new/docs/supported-models/multimodal_language_models.mdx` | `## Supported models` | Multimodal Language Model |
| `docs_new/docs/supported-models/embedding_models.mdx` | `## Supported Models` | Embedding Model |
| `docs_new/docs/supported-models/reward_models.mdx` | `## Supported models` | Reward Model |
| `docs_new/docs/supported-models/rerank_models.mdx` | `## Supported rerank models` | Rerank Model |
| `docs_new/docs/supported-models/diffusion_language_models.mdx` | `## Supported Models` | Diffusion Language Model |

> **注意：** `classify_models.mdx` 使用列表格式（非 HTML/Markdown 表格），不在扫描范围内。如果将来该文件改为表格格式，需在此表中补充。

```bash
git diff <last_commit>..<current_head> -- <file_path>
```

**查找内容分为两种表格格式：**

#### HTML 表格（5 个文件使用）

在 `<tbody>` 中新增的 `<tr>` 块（`+` 开头的行）。**各文件的 `<td>` 列数不同（3 或 4 列），但模型族始终在第 1 个 `<td>`，模型名称（HF 标识符）始终在第 2 个 `<td>`，只需提取这两列，忽略其余列。**

> **注意：** `multimodal_language_models.mdx` 包含多个独立的 `<table>` 块（如 Audio Transcription、Video Input Support），每个 `<table>` 都有自己的 `<tbody>`。所有 `<tbody>` 中新增的 `<tr>` 都应提取，统一归为 Multimodal Language Model 类型。

| 输出列 | 来源 `<td>` |
|---|---|
| 模型族 | 第 1 个 `<td>` — 去掉 `**...**` 加粗标记，保留括号前的族名 |
| 模型名称 | 第 2 个 `<td>` — 提取反引号或 `<code>` 标签中的 HuggingFace 标识符，多个标识符用逗号分隔 |
| 模型类型 | 由文件来源决定（见上表） |

#### Markdown 表格（`reward_models.mdx` 使用）

`reward_models.mdx` 使用 Markdown 表格（`| ... |` 格式），没有 `<tbody>` 标签。查找新增的表格数据行（`+` 开头的 `| ... |` 行），需排除表头分隔行（包含 `---`）。

对于 Markdown 表格行：
- 模型族：第 1 列 — 去掉 `**...**` 加粗标记
- 模型名称：第 2 列 — 提取反引号中的 HuggingFace 标识符
- 模型类型：Reward Model（由文件来源决定）

#### 边界情况

- 如果一行是被修改的（非纯新增），不算新增模型 — 仅报告纯新增的行（HTML `<tr>` 或 Markdown 表格行）。
- 如果模型类型不属于 6 种标准类型，根据文件所属类型归类。
- 注意 `<td>` 可能跨多行（元素内部有换行），解析时需将跨行 `<td>...</td>` 合并为一行处理。

### 5. 生成 history 文件

**文件名：** `YYYYMMDD.md`（当天日期），放在 `work_dirs/sglang-weekly/history/` 下。

1. 先检查 `work_dirs/sglang-weekly/history/` 目录是否存在，不存在则创建。
2. 生成文件内容。

**格式：**

```markdown
## 新增参数

| 前一个参数/新参数分组 | 参数名称 | 默认值 | 可选值 | 描述 |
|---|---|---|---|---|
| <分组或--prev> | --xx-yy | <默认值> | <可选值或 Type: xx> | <描述> |
```

**首列填写规则与示例：**

表格第一列用于定位新增参数的位置，有两种填法：

- **新分组：** 当新增参数上方有 `# Xxx options` 样式的注释行时，说明这是一个新的参数分组。在首列填入分组名称（如 `Http Server（新分组）`），参数名称为该分组下的第一个参数。
- **前一个参数：** 当无注释行时，填入 diff 中该参数前一个已有参数的 `--xx-yy` 名称，表示新增参数紧跟在该已有参数之后。

示例对照：

| 前一个参数/新参数分组 | 参数名称 | 默认值 | 可选值 | 描述 |
|---|---|---|---|---|
| Http Server（新分组） | --host | 127.0.0.1 | Type: str | HTTP 服务监听地址 |
| --grpc-mode | --skip-server-warmup | False | Type: bool（set to enable） | 跳过服务预热 |

```markdown
## 新增模型

| 模型类型 | 模型族 | 模型名称 |
|---|---|---|
| <模型类型> | <模型族> | <HF标识符> |
```

> 模型类型共分为 Large Language Model，Multimodal Language Model，Embedding Model，Reward Model，Rerank Model，Diffusion Language Model 这 6 种类型。

如果没有新增参数或没有新增模型，则省略对应章节（不输出空表格）。**如果两者都没有**（本周无任何新增），则不生成 history 文件，不更新 CHANGELOG.md，向用户报告"本周无新增参数和模型"后结束流程。

### 6. 验证生成的 history 文件

在写入文件后、提交前，进行以下检查：

- [ ] 文件名日期格式正确：`YYYYMMDD.md`，日期为当天。
- [ ] 表格不包含空行（无数据的占位行）。
- [ ] 参数名称无重复（两个新增参数不应有相同的 `--xx-yy`）。
- [ ] 默认值列不为空（至少填写 `Type: xx` 或 `（computed at runtime）`）。
- [ ] commit SHA 为完整 40 位。
- [ ] 如果本次有新增参数，`## 新增参数` 表格至少有一行数据。
- [ ] 如果本次有新增模型，`## 新增模型` 表格至少有一行数据。
- [ ] 如果既无新增参数也无新增模型，确认未生成 history 文件且未更新 CHANGELOG.md。
- [ ] 表格 Markdown 语法正确（分隔行 `|---|---|---|` 与列数匹配）。

### 7. 内容二次确认

格式验证通过后，**回溯源文件**对表格中的每一条数据进行准确性确认。此步骤在提交前执行，防止 diff 解析错误或遗漏导致写入错误数据。

**新增参数确认：**

对 `## 新增参数` 表格中的每一行：

1. 在 `python/sglang/srt/server_args.py` 中搜索该参数名称（`--xx-yy`），确认 `add_argument` 调用确实存在于当前 HEAD 中。
2. 核对默认值列：读取对应的 `default=` 值，与表格中填写的是否一致。如果默认值引用了 `ServerArgs.xxx` 或外部变量，确认解析结果正确。
3. 核对类型列：确认 `type=` 或 `action=` 与表格记录一致（如 `action="store_true"` 应记录为 `Type: bool（set to enable）`）。
4. 核对可选值列：如果 `choices=` 存在，确认解析后的实际值列表正确。
5. 核对描述列：确认 `help=` 文本与表格记录一致（允许合理截断，但不应出现编造内容）。
6. 核对首列定位：确认"前一个参数/新参数分组"与实际代码中的位置关系正确。

**新增模型确认：**

对 `## 新增模型` 表格中的每一行：

1. 在对应的文档文件（`docs_new/docs/supported-models/` 下的 `.mdx` 文件）中搜索该模型名称，确认模型条目确实存在于当前 HEAD 中。
2. 核对模型族：确认表格中的模型族名称与文档中的一致（去掉 `**...**` 标记后比对）。
3. 核对模型名称：确认 HuggingFace 标识符与文档中的一致，包括大小写和连字符。
4. 核对模型类型：确认类型分类与文件来源对应（如来自 `generative_models.mdx` 应为 Large Language Model）。

**结果处理：**

- 如果所有条目均确认无误，继续执行后续步骤。
- 如果发现不一致，**先尝试自行修正**：
  1. 回溯源文件重新读取对应代码/文档，定位正确值。
  2. 将表格中的错误条目更正为源文件中的实际值。
  3. 修正完成后重新执行一次该条目的确认核对，确保已修复。
- 仅在以下情况才暂停流程并提示用户确认：
  - 源文件中无法找到对应条目（可能是 diff 解析层级错误或文件路径变更）。
  - 同一条目反复修正仍不一致。
  - 差异项数量超过总条目的 30%，说明提取流程可能存在系统性问题，建议用户重新执行整个提取流程。

### 8. 更新 CHANGELOG.md

在 `work_dirs/sglang-weekly/CHANGELOG.md` 末尾追加一行：

```
YYYYMMDD.md: <current_head 完整 40 位 commit SHA>
```

**规则：**
- 仅追加，不允许修改或删除已有行。
- 使用完整的 40 位 commit SHA（`git rev-parse HEAD` 默认输出完整 SHA，不要截断）。

### 9. 提交并推送

```bash
cd work_dirs/sglang-weekly
git add history/YYYYMMDD.md CHANGELOG.md
git commit -m "Add weekly changelog for YYYY-MM-DD"
git push
```

## 常见错误

| 错误 | 正确做法 |
|---|---|
| diff 前忘记 `git pull` sglang 主线 | 先在 sglang 仓库执行 `git pull` |
| CHANGELOG 中使用短 commit id | 始终使用 `git rev-parse HEAD` 输出的完整 40 位 SHA |
| 将修改的参数当作新增参数 | 仅提取新增 `parser.add_argument` 的 `+` 行，不包括默认值或 help 文本的修改 |
| 未解析默认值/可选值中的变量引用 | 始终解析 `ServerArgs.xxx` 和其他变量引用到实际值 |
| 将模型修改当作新增模型 | 仅报告纯新增的表行，不包括修改的行 |
| 修改 CHANGELOG 已有行 | 只能追加，不允许删改 |
| 忽略 `reward_models.mdx` 的新增模型 | 5 个文件用 HTML `<tbody>` 表格，`reward_models.mdx` 用 Markdown 表格，需分别处理 |
| 未处理 `default_factory` 和 `__post_init__` | 遇到动态默认值时标注 `（computed at runtime）` 或 `（set in __post_init__）`，不可强行猜测 |
| 未验证输出就提交 | 始终先执行步骤 6 的格式 checklist 和步骤 7 的内容二次确认 |
| 跳过内容二次确认直接提交 | 必须执行步骤 7 回溯源文件核对每条数据，发现差异时暂停并提示用户 |
| `history/` 目录不存在导致 git add 失败 | 生成文件前先确保 `history/` 目录存在 |
| 本周无变更时仍生成空 history 文件 | 如果无新增参数且无新增模型，跳过整个流程，不生成文件、不更新 CHANGELOG |
