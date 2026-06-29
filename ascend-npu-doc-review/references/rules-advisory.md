# 建议级别规则（Advisory）

应尽量满足，不满足不阻止合入但应提示优化。输出报告中统一标记为 💡 SUGGESTION。

**规则引用编号**：A-1 ~ A-5

## 规则速查表

| 编号  | 规则名称                      | 严重级别      | 简要说明                                                                          |
|-------|-------------------------------|---------------|-----------------------------------------------------------------------------------| 
| A-1   | MDX 块级元素间用空行隔开      | 💡 SUGGESTION | 组件（`<Tabs>`、`<Warning>` 等）与 Markdown 内容之间、相邻块级组件之间用空行隔开  |
| A-2   | 行宽尽量不超过 120 字符       | 💡 SUGGESTION | 每行尽量 ≤120 字符，代码块、长链接、表格行除外                                    |
| A-3   | 服务配置命令参数过少提醒      | 💡 SUGGESTION | 启服命令中 env 数量或 CLI 参数数量 < 3 时提醒检查是否有遗漏                        |
| A-4   | 含 PROFILE/PROFILING 的环境变量确认 | 💡 SUGGESTION | `best_practice/`、`model-tutorials/` 下脚本中任何含 `PROFILE`/`PROFILING` 的 env 须确认是否属调试变量 |
| A-5   | 拼写与语法检查                      | 💡 SUGGESTION | 对变更的正文内容（非代码块）进行拼写和基础语法检查                                  |

---

## A-1：MDX 块级元素间用空行隔开

**检查范围**：所有 `.mdx` 文件。

**规则**：Markdown 内容与 MDX / JSX 组件之间、以及块级组件之间，都需要用空行隔开，避免渲染异常。

**适用场景**：

| 场景 | 要求 |
|------|------|
| Markdown 段落与 `<Tabs>` / `<Tab>` / `<Warning>` / `<Note>` / `<Tip>` 等组件相邻 | 之间空一行 |
| 组件闭合标签（`</Tab>`、`</Warning>` 等）与后续 Markdown 内容 | 之间空一行 |
| 相邻的两个块级组件（如 `</Tab>` 后紧跟 `<Warning>`） | 之间空一行 |
| 代码块（```）与前后内容 | 前后各空一行（已有 Markdown 规范） |

**示例（合规）**：
```mdx
<Tip>
This is a tip.
</Tip>

Next paragraph starts here with a blank line above.

<Tabs>
  <Tab title="Tab A">
Content A
  </Tab>

  <Tab title="Tab B">
Content B
  </Tab>
</Tabs>

Following content after tabs.
```

**示例（不合规）**：
```mdx
</Tip>
Next paragraph — no blank line separator.
```

**检查方式**：
1. 搜索组件闭合标签后紧跟非空行、非代码块结尾的行
2. 搜索组件开放标签前紧邻非空行的内容行

**修复指引**：在块级组件与周边 Markdown 之间补空行。

---

## A-2：行宽尽量不超过 120 字符

**检查范围**：所有 `.mdx` 文件。

**规则**：每行字符数尽量控制在 120 字符以内。以下情况可以例外：

| 例外场景 | 说明 |
|----------|------|
| 超长 URL / 链接 | 避免链接被截断导致失效 |
| 代码块内内容 | 代码块内不适用此规则 |
| 表格行 | 表格行结构优先 |
| MDX 组件标签 | 如 `<Accordion title="...">` 等不便拆分的组件 |
| 连续的单词 / 标识符 | 不可拆分且超过 120 字符 |

**检查方式**：
1. 逐行计算字符数（不含行尾换行符）
2. 标记超过 120 字符的行
3. 排除代码块、表格、长链接、MDX 组件标签等例外情形
4. 对剩余可拆分的超长行给出建议

**修复指引**：在不影响语义的前提下，将长段落拆分为多行；或将长句拆分为多个短句。

---

## A-3：服务配置命令参数过少提醒

**检查范围**：包含完整服务配置命令的代码块（`python3 -m sglang.launch_server` 或 `sglang serve`）。

**规则**：当脚本块包含启服命令及其附属环境变量时，若环境变量或命令行参数数量过少，应提醒作者参考最佳实践检查是否有遗漏。

- 如果当前代码块中 `export` 环境变量的数量 **< 3**，应提醒检查是否有缺失的环境变量
- 如果启服命令中配置型命令行参数的计数（不含基础设施类参数）**< 3**，应提醒检查配置是否不完整

不计入 CLI 参数计数的基础设施类通用参数：`--host`、`--port`、`--log-level`、`--log-level-http`、`--api-key`、`--trust-remote-code`。其余模型/性能类参数（如 `--tp-size`、`--mem-fraction-static`、`--max-running-requests` 等）应计入。

**检查方式**：
1. 定位代码块中包含 `sglang.launch_server` 或 `sglang serve` 的完整命令行
2. 统计同一代码块内 `export` 语句的数量（`VAR=val command` 行内赋值形式也计入）
3. 统计启服命令行中配置型参数的数量（排除基础设施类通用参数）
4. env < 3 或 params < 3 时给出建议

**修复指引**：参考 `best_practice/deepseek_r1.mdx` 中同类部署场景的 `export` 和启服命令，检查是否有遗漏的必要环境变量或配置参数。

---

## A-4：含 PROFILE / PROFILING 的环境变量确认

**检查范围**：`best_practice/` 和 `model-tutorials/` 目录下 `.mdx` 文件中的脚本代码块。

**规则**：对于 M-F-9 未覆盖的、名称中包含 `PROFILE` 或 `PROFILING`（不区分大小写）的环境变量，应提醒作者确认是否属于调试用途。属于调试用途的变量不应出现在面向用户的部署脚本中。

M-F-9 已覆盖的变量（`SGLANG_NPU_PROFILING`、`SGLANG_NPU_PROFILING_STAGE`、`SGLANG_PROFILE_WITH_STACK`）直接按 M-F-9 处理，不重复触发本规则。

**检查方式**：
1. 判断当前变更文件是否属于 `best_practice/` 或 `model-tutorials/` 目录
2. 在该文件的脚本代码块中搜索名称含 `PROFILE` 或 `PROFILING`（不区分大小写）的环境变量
3. 排除已在 M-F-9 中明确禁止的三个变量
4. 发现任一项即标记为建议，提醒作者确认

**修复指引**：如确认该变量为调试用途，从部署脚本中移除，可在正文中另起独立段落说明如何开启调试。如确认为部署所需变量（非调试），可忽略此建议。

---

## A-5：拼写与语法检查

**检查范围**：所有变更 `.mdx` 文件中，diff 的 **新增或修改行**（`+` 前缀行）里的正文内容，不包含代码块（```...```）内的内容。

**规则**：对变更的英文正文进行拼写和基础语法检查。以下场景不适用此规则：

| 排除场景 | 说明 |
|---|---|
| 代码块内内容（```...```） | 脚本/命令中的标识符、参数名可能有非标准拼写 |
| 专业术语 / 缩写 / 品牌名 | 如 `SGLang`、`Ascend`、`NPU`、`HCCL`、`NNAnalyzer` 等 |
| YAML frontmatter | 标题和 description 中的固定用语 |
| 文件路径 / URL | 不做拼写检查 |
| MDX 组件标签及属性 | 如 `<Tabs>`、`<Tab title="...">`、`<Warning>` 等 |

**主要检查项**：

| 类别 | 示例 |
|---|---|
| 明显拼写错误 | `recieve` → `receive`，`enviornment` → `environment`，`configration` → `configuration` |
| 重复单词 | `the the`、`to to` |
| 主谓不一致 | `The model were tested` → `The model was tested` |
| 冠词缺失/多余 | `in Ascend NPU` → `on the Ascend NPU`（视上下文） |
| 常见混淆词 | `it's` / `its`，`their` / `there`，`affect` / `effect` |

**检查方式**：
1. 从 PR diff 中提取每个 `.mdx` 文件的新增/修改行
2. 排除代码块（```...```）、frontmatter、URL、组件标签内的内容
3. 对剩余英文正文逐行检查拼写和基础语法
4. 不要求零误报——本规则为建议级别，有明显错误时报告，不确定时跳过

**修复指引**：按标准英语拼写和语法修正。专业术语和品牌名保持原样。
