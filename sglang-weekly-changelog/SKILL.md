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

**目标文件：** `python/sglang/srt/server_args.py` 及 `python/sglang/srt/arg_groups/arg_utils.py`

```bash
git diff <last_commit>..<current_head> -- python/sglang/srt/server_args.py
```

**背景：** 当前 server_args.py 中，绝大多数参数通过 `ServerArgs` dataclass 的字段注解 `A[T, Arg(...)]` 定义（由 `add_cli_args_from_dataclass()` 自动生成 argparse），仅少数特殊参数（deprecated 重定向、动态 choices、`--config` 元参数）仍在 `add_cli_args()` 方法中手动调用 `parser.add_argument()`。因此提取新增参数需同时扫描两类位置。

#### 3a. 主来源 — `ServerArgs` dataclass 字段（`A[T, Arg(...)]` 注解）

在 diff `+` 行中查找新增的 dataclass 字段定义，格式为：

```python
field_name: A[T, "bare help string"] = <default_value>
field_name: A[T, Arg(help="...", choices=..., aliases=[...], ...)] = <default_value>
```

**字段映射表（`A[T, Arg(...)]` 模式）：**

| 字段 | 来源 | 说明 |
|---|---|---|
| 参数名称 | `Arg(cli_name=...)` 或 `Arg(aliases=[...])`，否则由字段名自动推导（`field_name` → `--field-name`） | 保留 `--xx-yy` 格式；若有 aliases，主名 + 别名均列出 |
| 类型 | 类型注解 `T` | `str`、`int`、`float`、`bool`（auto `store_true`）等；`Optional[X]` 表示可选；`Literal["a","b"]` 自动生成 choices |
| 默认值 | 字段赋值 `= VALUE` | 紧跟字段定义行末尾的值；若为 `dataclasses.field(default_factory=...)` 见下方说明 |
| 可选值 | `Arg(choices=...)` 或 `Literal["a","b"]` 自动推导 | 可能引用变量，需解析变量值 |
| 描述 | `Arg(help=...)` 或裸字符串 | 多行字符串需合并为一整句 |
| 分组/前一个参数 | 字段上方的注释 banner `# ---- Xxx ----`，或前一字段的参数名 | 出现 banner → 新分组；否则 → 前一个字段的 `--xx-yy` |

#### 3b. 次来源 — `add_cli_args()` 方法中的 `parser.add_argument()` 手动条目

在 `add_cli_args()` 静态方法（约第 6448 行起）中，diff `+` 行可能出现新的 `parser.add_argument(...)` 调用。此处的参数分两类：

1. **新功能参数**（如 dynamic choices 参数或新 deprecated alias）：正常提取，映射表同旧版规则：
   - 参数名称：第一个位置参数（`"--xx-yy"`）
   - 类型：`type=` 关键字
   - 默认值：`default=` 关键字
   - 可选值：`choices=` 关键字
   - 描述：`help=` 关键字

2. **旧参数改造为 deprecated**：旧 commit 中已存在的参数，仅修改了 `action=` 为 deprecated action 或增加了 `new_flag=`。**不算新增，不收录。**

#### 3c. 新旧参数的判定规则

对每个候选新增参数，按以下规则判断是否应收录：

1. **纯新增参数**：diff 中出现新的 dataclass 字段（`A[T, ...]`）或新的 `parser.add_argument(...)` `+` 行，且旧 commit 中不存在同名参数。**必须收录。**
2. **旧参数变为 deprecated**：旧 commit 中已存在该参数，diff 中仅修改了其 action 或增加了 `new_flag=` 字段。**不算新增参数，不收录。**
3. **新增的 deprecated alias**：旧 commit 中不存在该参数名，但 diff 中该参数一出现就已带有 deprecated action（如 `action=DeprecatedAliasStoreAction`，或 dataclass 字段中 `Arg(action=DeprecatedStoreTrueAction, ...)`）。**作为 CLI flag 是新增的，应收录。** 在描述列中标注 `[Deprecated alias for --xx-yy]`。默认值列填写该 alias 指向新参数的默认值（解析方式同普通参数）；若无明显指向的默认值，标注 `Same as --new-flag (deprecated)`。

**判断方法：** 对每个候选参数名，先用 `git show <last_commit>:python/sglang/srt/server_args.py | Select-String '<参数名>'` 确认旧 commit 中是否存在该参数名，再决定属于上述哪种情况。

#### 3d. 默认值解析细节

- **dataclass 字段直接赋值：** 当字段定义为 `field_name: A[str, ...] = "auto"` 时，默认值为 `"auto"`；当定义为 `field_name: A[bool, ...] = False` 时，类型为 `bool flag (set to enable)`，默认值为 `False`。
- **`dataclasses.field(default_factory=...)`：** 默认值在运行时动态计算，无法静态解析。**输出时标注 `(computed at runtime: <factory expression>)`，并给出 factory 表达式内容作为参考。**
- **`__post_init__` 中动态修改：** 即使字段有静态默认值，`__post_init__` 方法可能在运行时覆盖它。**若确认值在 `__post_init__` 中动态计算，输出时标注 `(set in __post_init__, initial value: xxx)`。**
- **默认值引用变量：** 如果默认值引用了其他变量（如 `default=ServerArgs.xxx` 或 `default=SOME_CONSTANT`），需解析该变量的实际值。

#### 3e. 可选值解析细节

- 当 `choices=LOAD_FORMAT_CHOICES` 时，需在文件顶部查找该变量的定义，列出实际的 choice 值。
- 当 `choices=[m.name.lower() for m in RealKvHashMode]` 时，需解析该枚举/类，列出实际的 choice 值。
- 当类型注解为 `Literal["a", "b", "c"]` 时，自动生成的 choices 为 `a, b, c`。

**可选值格式化规则：** 可选值以逗号分隔的纯文本列出，不使用方括号 `[]` 包裹（即 `a, b, c` 而非 `[a, b, c]`）。类型为字符串的可选值不使用引号包裹（即 `none, log, raise` 而非 `"none", "log", "raise"`）。

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
|---|---|---|---|---|---|
| <group or --prev> | --xx-yy | <default> | <choices or Type: xx> | <description> |
```

**首列填写规则与示例：**

表格第一列用于定位新增参数的位置，有两种填法：

- **新分组：** 当新增参数上方有 `# ---- Xxx ----` 样式的 banner 注释行时（如 `# ---- HTTP server ----`），说明这是一个新的参数分组。在首列填入分组名称（如 `HTTP server (new group)`），参数名称为该分组下的第一个参数。
- **前一个参数：** 当无 banner 注释行时，填入 diff 中该参数前一个已有参数的 `--xx-yy` 名称，表示新增参数紧跟在该已有参数之后。

示例对照：

| 前一个参数/新参数分组 | 参数名称 | 默认值 | 可选值 | 描述 |
|---|---|---|---|---|---|
| Http Server (new group) | --host | 127.0.0.1 | Type: str | HTTP server listen address |
| --grpc-mode | --skip-server-warmup | False | bool flag (set to enable) | Skip server warmup |

```markdown
## 新增模型

| 模型类型 | 模型族 | 模型名称 |
|---|---|---|
| <model type> | <model family> | <HF identifier> |
```

> 模型类型共分为 Large Language Model，Multimodal Language Model，Embedding Model，Reward Model，Rerank Model，Diffusion Language Model 这 6 种类型。

**Markdown 特殊字符转义：**

表格单元格内的文本可能包含 Markdown/HTML 特殊字符，需要在写入文件前进行转义，否则渲染时会丢失内容：

| 特殊字符 | 转义方式 | 说明 |
|---|---|---|
| `<` 和 `>` | 用反斜杠转义为 `\<` 和 `\>`，或用反引号包裹为 `` `<...>` `` | 未转义的 `<...>` 会被当作 HTML 标签吞掉。注意：仅对独立出现的 `<` `>` 转义；`>=` `<=` `->` 等复合运算符无需转义 |
| `\|` | 用反斜杠转义为 `\\|` | 未转义的 `\|` 会被当作表格列分隔符 |
| `__text__` | 用反引号包裹为 `` `__text__` `` | 双下划线 `__...__` 会被部分 Markdown 渲染器解析为粗体标记，导致内容丢失或格式错乱 |

转义范围覆盖所有表格列（描述、默认值、可选值、模型名称等）。转义后再写入文件。

**表格内容语言规则：** 最终输出的表格中，表头可以为中文，但所有数据单元格（参数描述、默认值标注、分组名称等）必须使用英文，不得包含中文。例如：默认值列的标注应使用英文括号和英文术语（如 `(computed at runtime)` 而非 `（computed at runtime）`，`(set in __post_init__, initial value: xxx)` 而非 `（set in __post_init__，初始值: xxx）`）；分组名称应写作 `(new group)` 而非 `（新分组）`；废弃标注应写作 `(deprecated)` 而非 `（废弃）`。模型名称和 HuggingFace 标识符本身不受此限制（通常不含中文）。

如果没有新增参数或没有新增模型，则省略对应章节（不输出空表格）。**如果两者都没有**（本周无任何新增），则不生成 history 文件，不更新 CHANGELOG.md，向用户报告"本周无新增参数和模型"后结束流程。

### 6. 验证生成的 history 文件

在写入文件后、提交前，进行以下检查：

- [ ] 文件名日期格式正确：`YYYYMMDD.md`，日期为当天。
- [ ] 表格不包含空行（无数据的占位行）。
- [ ] 参数名称无重复（两个新增参数不应有相同的 `--xx-yy`）。
- [ ] 默认值列不为空（至少填写 `Type: xx` 或 `(computed at runtime)`）。
- [ ] commit SHA 为完整 40 位。
- [ ] 如果本次有新增参数，`## New Parameters` 表格至少有一行数据。
- [ ] 如果本次有新增模型，`## New Models` 表格至少有一行数据。
- [ ] 如果既无新增参数也无新增模型，确认未生成 history 文件且未更新 CHANGELOG.md。
- [ ] 表格 Markdown 语法正确。参数表为 5 列，分隔行应为 `|---|---|---|---|---|`；模型表为 3 列，分隔行应为 `|---|---|---|`。列数必须与表头匹配。
- [ ] 表格内容中无未转义的 `<...>` 或裸 `|`（会在渲染时丢失内容）。
- [ ] 表格中数据单元格内容均为英文，不含中文（表头可为中文，但描述、默认值、标注等数据单元格不得出现中文）。

### 7. 内容二次确认

格式验证通过后，**回溯源文件**对表格中的每一条数据进行准确性确认。此步骤在提交前执行，防止 diff 解析错误或遗漏导致写入错误数据。

**新增参数确认：**

对 `## 新增参数` 表格中的每一行：

1. 在 `python/sglang/srt/server_args.py` 中搜索该参数名称（`--xx-yy`），确认其定义确实存在于当前 HEAD 中。
   - 若参数来自 dataclass 字段（`A[T, Arg(...)]`），确认字段存在且类型/注解正确。
   - 若参数来自 `add_cli_args()` 中的 `parser.add_argument()`，确认调用正确。
2. 核对默认值列：
   - dataclass 字段：读取 `= VALUE`，与表格一致。
   - `parser.add_argument()`：读取 `default=` 值。
   - 若引用 `ServerArgs.xxx` 或外部变量，确认解析结果正确。
3. 核对类型列：
   - dataclass 字段：类型注解 `T` 决定（`bool` → `bool flag (set to enable)`，`int` → `Type: int`，`str` → `Type: str` 等）。
   - `parser.add_argument()`：确认 `type=` 或 `action=` 与表格记录一致（如 `action="store_true"` → `bool flag (set to enable)`）。
4. 核对可选值列：
   - dataclass 字段：检查 `Arg(choices=...)` 或 `Literal[...]` 自动推导的 choices。
   - `parser.add_argument()`：检查 `choices=`。
   - 确认解析后的实际值列表正确。
5. 核对描述列：
   - dataclass 字段：检查 `Arg(help=...)` 或裸字符串内容。
   - `parser.add_argument()`：检查 `help=` 内容。
   - 必须完整匹配，不应出现截断或编造内容；注意已转义的 `\<` `\>` 对应源文件中的 `<` `>`。
6. 核对首列定位：
   - dataclass 字段：确认 `# ---- Xxx ----` comment banner 或前一字段的参数名与实际代码位置一致。
   - `parser.add_argument()`：确认前一个参数或分组名称正确。

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
| 只扫描 `add_cli_args()` 中的 `parser.add_argument` 而忽略 dataclass 字段 | 绝大多数新增参数现在以 `A[T, Arg(...)]` 注解的 dataclass 字段形式定义，必须同时扫描这类字段中的新增行 |
| 只扫描 dataclass 字段而忽略 `add_cli_args()` 中的手动条目 | 少数参数（dynamic choices、deprecated alias、--config）仍在 `add_cli_args()` 中手动定义，也需扫描 |
| 将修改的参数当作新增参数 | 仅提取 diff 中纯新增的 dataclass 字段或 `parser.add_argument` 调用（`+` 行），不包括对已有参数默认值/help 文本的修改 |
| 将 `no_cli=True` 的字段当作 CLI 参数收录 | `Arg(no_cli=True)` 表示该字段无 CLI surface（仅 Python 内部使用），不应加入参数表格 |
| 漏掉新增的 deprecated alias 参数 | 旧 commit 中不存在的参数名，即使出现时就带 `DeprecatedAliasStoreAction` 等 action，或 dataclass 字段中 `Arg(action=DeprecatedStoreTrueAction, ...)`，也应收录（标注 `[Deprecated alias for --xx-yy]`）。区分方法：用 `git show <last_commit>:<file> \| Select-String '<参数名>'` 确认旧 commit 是否已有该参数 |
| 未解析默认值/可选值中的变量引用 | 始终解析 `ServerArgs.xxx` 和其他变量引用到实际值 |
| 将模型修改当作新增模型 | 仅报告纯新增的表行，不包括修改的行 |
| 修改 CHANGELOG 已有行 | 只能追加，不允许删改 |
| 忽略 `reward_models.mdx` 的新增模型 | 5 个文件用 HTML `<tbody>` 表格，`reward_models.mdx` 用 Markdown 表格，需分别处理 |
| 未处理 `default_factory` 和 `__post_init__` | 遇到动态默认值时标注 `(computed at runtime)` 或 `(set in __post_init__)`，不可强行猜测 |
| 未验证输出就提交 | 始终先执行步骤 6 的格式 checklist 和步骤 7 的内容二次确认 |
| 跳过内容二次确认直接提交 | 必须执行步骤 7 回溯源文件核对每条数据，发现差异时暂停并提示用户 |
| `history/` 目录不存在导致 git add 失败 | 生成文件前先确保 `history/` 目录存在 |
| 本周无变更时仍生成空 history 文件 | 如果无新增参数且无新增模型，跳过整个流程，不生成文件、不更新 CHANGELOG |
