# 重要级别规则（Mandatory）

必须全部通过，不通过则阻止合入。分为两类：

| 类别 | 编号 | 说明 | 违反后果 | 严重级别 |
|------|------|------|----------|----------|
| **正确性** | M-C-x | 影响脚本/命令的运行时正确性 | 用户按文档操作会直接报错 | 🔴 BLOCK |
| **重要格式** | M-F-x | 影响文档规范性和可维护性 | 用户可能困惑，但脚本能跑通 | 🟡 ISSUE |

> 两类规则均为"必须修复才可合入"。区别仅在输出报告的严重级别标记：M-C-x → 🔴 BLOCK，M-F-x → 🟡 ISSUE（与 SKILL.md 的严重级别映射保持一致）。

**规则总数**：正确性 1 条，重要格式 13 条。

## 规则速查表

| 编号  | 规则名称                      | 严重级别 | 简要说明                                                                   |
|-------|-------------------------------|----------|----------------------------------------------------------------------------|
| M-C-1 | 含分号值的环境变量须加引号    | 🔴 BLOCK | `export` 值含 `;` 时必须用双引号包裹，否则 shell 会将分号解释为命令分隔符  |
| M-F-1 | 脚本开头须有动态字段注释      | 🟡 ISSUE | 脚本代码块开头必须有 `#` 注释块列出需替换的变量/占位符                     |
| M-F-2 | 动态字段占位符格式统一        | 🟡 ISSUE | 占位符格式须与 `best_practice/deepseek_r1.mdx` 中同类字段保持一致          |
| M-F-3 | 脚本中不得硬编码 IP           | 🟡 ISSUE | 禁止硬编码非 `localhost` / `127.0.0.1` 的 IP，多节点场景必须使用占位符     |
| M-F-4 | 脚本中不得有重复字段          | 🟡 ISSUE | 同一代码块内环境变量、命令行参数、Shell 变量不得重复定义或赋值             |
| M-F-5 | 链接有效性校验                | 🟡 ISSUE | 站内链接须可解析指向目标，旧站 `docs.sglang.io` 链接应改写为 root-relative |
| M-F-6 | 删除文档时同步 docs.json      | 🟡 ISSUE | 删除 `.mdx` 后须同步清理 `navigation` 和 `redirects` 中对应条目            |
| M-F-7 | 新增文档时同步 docs.json      | 🟡 ISSUE | 新增 `.mdx` 后须在对应 group 的 `pages` 数组中添加条目                     |
| M-F-8 | 脚本中不得硬编码自定义路径    | 🟡 ISSUE | 模型/数据/日志等自定义路径须用变量替代（系统路径除外）                     |
| M-F-9 | 部署脚本中不得含调试环境变量  | 🟡 ISSUE | `best_practice/`、`model-tutorials/` 下脚本不得含 `SGLANG_NPU_PROFILING` 等调试变量 |
| M-F-10 | 用例内信息一致性              | 🟡 ISSUE | 同一用例的标题、元数据、速查表行、bench 命令输入输出须一致                        |
| M-F-11 | 新增文档须有完整前置元数据    | 🟡 ISSUE | 新增 `.mdx` 文件须以 YAML frontmatter 开头，含 `title` 和 `description`             |
| M-F-12 | 成对标点符号须正确闭合        | 🟡 ISSUE | `**`、`` ` ``、`"`、`'`、`[]()`、`【】` 等成对符号不得漏闭合，否则导致渲染异常        |
| M-F-13 | 单位使用须符合 SI 规范        | 🟡 ISSUE | 数字与单位间须有空格，单位符号大小写须正确（如 `10 ms`、`100 GB`），禁用非标准单位表示 |

---

# 正确性规则

---

## M-C-1：含分号值的环境变量须加引号

**检查范围**：所有脚本代码块中 `export` 语句，以及 `HCCL_ALGO` 等环境变量的赋值。

**规则**：当环境变量值中包含分号（`;`）时，必须使用双引号包裹整个值，否则 Shell 会将分号解释为命令分隔符，导致后续内容被当作独立命令执行，引发不可预期的错误。

**错误示例**：
```bash
export HCCL_ALGO=level0:NA;level1:ring
```
以上在 Shell 中等价于两条独立命令：
```bash
export HCCL_ALGO=level0:NA
level1:ring
```
`level1:ring` 会被 shell 当作命令执行，导致报错。

**正确示例**：
```bash
export HCCL_ALGO="level0:NA;level1:ring"
```

**检查方式**：
1. 搜索所有 `export` 语句中值含分号但未使用引号的行
2. 不限于 `HCCL_ALGO`，任何含分号值的环境变量赋值都适用此规则

**修复指引**：为含分号的环境变量值添加双引号包裹。

---

# 重要格式规则

---

## M-F-1：脚本开头须有动态字段注释

**检查范围**：所有包含脚本的代码块（语言标签为 `bash`、`shell` 或 `python` 的命令示例）。

**规则**：脚本代码块的开头必须有段落注释，列出运行前需要修改的动态字段。企业内置项不需要列出。

**参考模板**（来自 `best_practice/deepseek_r1.mdx`）：

```bash
# ============================================================
# Before running, update the following variables:
#   P_IP: prefill node IP address
#   D_IP: decode node IP address
#   ASCEND_MF_STORE_URL: prefill node IP with port
#   MODEL_PATH: path to the model weights directory
#   HCCL_SOCKET_IFNAME: network interface name for HCCL
#   GLOO_SOCKET_IFNAME: network interface name for Gloo
# ============================================================
```

**注释格式要求**：
- 使用 `# ============================================================` 分隔线开头和结尾
- 第一行说明：`# Before running, update the following variables:`（变量列表）或 `# Before running, replace the following placeholders:`（占位符列表）
- 每行一个字段，格式：`#   FIELD_NAME: description`
- 注释块与脚本内容之间需要空行隔开

**检查方式**：
1. 定位文件中所有代码块（```bash / ```shell）
2. 确认代码块开头紧邻位置有上述格式的注释块
3. 确认注释中列出的动态字段与脚本中实际使用的占位符一致、无遗漏

**修复指引**：参考 `best_practice/deepseek_r1.mdx` 中第一个脚本块的注释格式，补充或修正缺失的注释块。

---

## M-F-2：动态字段占位符格式统一

**检查范围**：文档中脚本和正文中所有需要用户自行替换的动态字段。

**规则**：
1. 先确认涉及的动态字段有哪些
2. 查看 `best_practice/deepseek_r1.mdx` 中是否已有同类字段的占位符，如有则沿用其格式
3. 如果最佳实践中不存在该类字段，可使用合适的占位符格式，但同一文档内须保持一致

**占位符格式参考**（IP / 接口 / 路径三类已在 `best_practice/deepseek_r1.mdx` 中使用）：

| 占位符格式 | 适用场景 | 示例 | 来源 |
|-----------|----------|------|------|
| `<your ...>` | IP 地址类 | `<your prefill ip>`、`<your decode ip1>` | 最佳实践已有 |
| `<network-interface>` | 网络接口类 | `<network-interface>` | 最佳实践已有 |
| `/path/to/model-weights` | 模型路径 | `/path/to/model-weights` | 最佳实践已有 |
| `<secret>` 或 `<...>` | 密钥/凭据 | `<secret>` | 推荐格式（最佳实践暂无同类字段，按需采用并在文档内保持一致） |

禁止混用不同的占位符风格表示同类含义（如 `/path/to/model` 和 `<model_path>` 在同一文档中混用）。

**检查方式**：
1. 遍历文档中所有角度括号 `<>` 和路径型占位符
2. 将占位符按语义类别分组（IP 类、路径类、接口类、密钥类）
3. 确认同类别占位符格式一致，且与最佳实践对齐

**修复指引**：将不一致的占位符改为与最佳实践中同类字段一致的格式。

---

## M-F-3：脚本中不得硬编码 IP

**检查范围**：所有脚本代码块中的 IP 地址。

**规则**：
- 禁止硬编码除 `localhost` / `127.0.0.1` 之外的任何 IP 地址
- `localhost` 和 `127.0.0.1` 在以下场景可不处理：
  - 本机部署脚本（如 `--host 127.0.0.1 --port 6688` 单节点服务）
  - 本机 benchmark 请求脚本（如 `--host 127.0.0.1 --port 6688`）
  - Docker 端口映射中的本地地址
- 多节点部署场景（如 PD Disaggregation）中涉及节点间通信的 IP 必须使用占位符

**示例（合规）**：
```bash
# 单节点本机部署 — 127.0.0.1 可接受
python3 -m sglang.launch_server --host 127.0.0.1 --port 6688 ...

# 多节点部署 — 必须使用占位符
P_IP=('<your prefill ip>')
D_IP=('<your decode ip>')
```

**检查方式**：
1. 提取脚本中所有 IP 地址
2. 过滤出非 `localhost` / `127.0.0.1` 的 IP
3. 确认每个此类 IP 是否应在多节点场景下改为占位符

**修复指引**：将硬编码的多节点通信 IP 替换为最佳实践中对应的占位符格式。

---

## M-F-4：脚本中不得有重复字段

**检查范围**：所有脚本代码块。

**规则**：同一代码块内，环境变量、命令行参数、Shell 变量不得重复定义或赋值。重复不仅指字面完全一致，也包括语义相同的重复。

**示例（不合规）**：
```bash
export HCCL_SOCKET_IFNAME=eth0
# ... 中间代码 ...
export HCCL_SOCKET_IFNAME=eth1   # 重复定义，应只保留最终需要的值，或使用变量
```

**检查方式**：
1. 提取每个代码块中所有赋值语句（`export`、`VAR=`、`--flag value`）
2. 检测同名字段是否存在多次赋值
3. 检查同一 shell 脚本块中同一 flag 是否多次出现（如两个 `--tp-size`）

**修复指引**：删除多余的重复定义/赋值，只保留正确的那个。如果确实需要不同值，应明确注释说明原因。

---

## M-F-5：链接有效性校验

**检查范围**：文档中所有链接。

**规则**：

| 链接类型 | 要求 |
|----------|------|
| 站内链接 — 绝对路径（root-relative） | 必须以 `/docs` 开头，目标文件/锚点必须存在；这是新站（Mintlify）推荐写法 |
| 站内链接 — 相对路径 | 可解析到目标文件，锚点必须存在 |
| 同文件锚点 | 文件内必须有匹配的 `id`、`name` 或 heading anchor |
| 旧站绝对链接（`https://docs.sglang.io/...`） | ⚠️ 指向待迁移的旧 Sphinx 站，应改写为 root-relative `/docs/...` 路径 |

**站内链接绝对路径格式**（合规）：
```
/docs/hardware-platforms/ascend-npus/best_practice/deepseek_r1#single-node-pd-mixed
```

禁止使用旧路径格式如 `/platforms/ascend/ascend_npu.html`，以及指向旧站的 `https://docs.sglang.io/...` 绝对链接（保留在 `redirects` 中可接受，但新文档正文链接须用 root-relative `/docs/...` 新路径）。

**检查方式**：
1. 提取文档中所有链接（Markdown 链接 `[text](url)`、JSX `<a href="">`）
2. 对每个链接分类（站内绝对 `/docs...` / 站内相对 / 同文件锚点 / 旧站 `docs.sglang.io` / 外部）
3. 对整个 `ascend-npus/` 目录做 Glob + Grep，验证站内跨文件、同文件锚点目标存在
4. 外部链接不做深度校验，但应检查 URL 格式正确且未失效（如指向 404 页面）；发现旧站绝对链接时建议改写为 root-relative `/docs/...`

**修复指引**：
- 链接无法解析：修正路径或锚点
- 路径不以 `/docs` 开头：补全完整路径

---

## M-F-6：删除文档文件时同步 docs.json

**检查范围**：PR 中包含 `.mdx` 文件删除操作时。

**规则**：删除 `ascend-npus/` 下的 `.mdx` 文件后，必须确认 `docs.json` 中相应的导航条目和 redirects 条目已同步处理：
1. `navigation` 中的对应页面条目已移除
2. `redirects` 中以该文件为目标的旧重定向已清理或更新

**检查方式**：
1. 从 PR diff 中识别被删除的 `.mdx` 文件
2. 将其路径转换为 `docs.json` 中的导航路径格式（去掉 `docs_new/` 前缀和 `.mdx` 后缀）
3. 在 `docs.json` 中搜索该路径，确认不再出现在 navigation 中

**修复指引**：在 `docs.json` 中移除对应的 navigation 条目和 redirects 条目。

---

## M-F-7：新增文档文件时同步 docs.json

**检查范围**：PR 中包含 `.mdx` 文件新增操作时。

**规则**：新增 `ascend-npus/` 下的 `.mdx` 文件后，必须确认 `docs.json` 中已添加相应的导航目录链接。`"group": "Ascend NPUs"` 下当前仅有两个子组（`Model Tutorials`、`Best Practice`），其余均为顶级 `pages`：

1. 顶级页面（直接位于 `ascend-npus/` 下）添加到 `"group": "Ascend NPUs"` 的 `pages` 数组中
2. `model-tutorials/` 下的文件添加到 `"group": "Model Tutorials"` 子组
3. `best_practice/` 下的文件添加到 `"group": "Best Practice"` 子组
4. 其他子目录（如 `diffusion/`）在 `"Ascend NPUs"` 下**目前没有对应子组**：以实际 `docs.json` 结构为准，将文件加入合适的现有分组，或新建子组并在同一 PR 中体现；不要假设存在某个未在 `docs.json` 中出现的子组

**检查方式**：
1. 从 PR diff 中识别新增的 `.mdx` 文件
2. 将其路径转为 `docs.json` 导航路径
3. 在 `docs.json` 对应 group 的 `pages` 中确认该路径已添加

**修复指引**：在 `docs.json` 的对应 group 中添加新的页面条目。

---

## M-F-8：脚本中不得硬编码自定义路径

**检查范围**：所有脚本代码块。

**规则**：脚本中不得硬编码自定义路径，包括但不限于：
- 模型路径：`/data/models/DeepSeek-R1` → 应使用变量 `MODEL_PATH=/path/to/model-weights`
- 草稿模型路径：`/data/models/eagle3-8b` → 应使用变量如 `DRAFT_MODEL_PATH=/path/to/draft-model`
- 专家热度分布文件路径：`/data/eplb/experts_distribution.csv` → 应使用变量如 `EXPERT_DIST_PATH=/path/to/experts-distribution`
- 日志/输出/缓存路径等自定义目录

系统路径（如 `/usr/local/Ascend/...`、`/sys/devices/...`）和临时文件路径不在此列。

**检查方式**：
1. 在脚本中搜索路径型字符串（以 `/` 开头且包含常规目录层级）
2. 排除系统路径（`/usr/`、`/sys/`、`/etc/`、`/proc/` 等）
3. 确认剩余自定义路径是否已用变量替代

**修复指引**：将硬编码的自定义路径替换为命名清晰的 Shell 变量，变量初始化值使用占位符路径如 `/path/to/model-weights`，并在脚本开头注释块中说明该变量。

---

## M-F-9：部署脚本中不得含调试环境变量

**检查范围**：`best_practice/` 和 `model-tutorials/` 目录下 `.mdx` 文件中的脚本代码块。

**规则**：以下调试/分析用途的环境变量不应出现在面向最终用户的部署脚本中，这些变量仅供开发调试使用，混入部署脚本会造成用户困惑且可能导致非预期的性能开销：

| 环境变量 | 用途 |
|----------|------|
| `SGLANG_NPU_PROFILING` | NPU profiling 开关，开启后采集 profiling 数据 |
| `SGLANG_NPU_PROFILING_STAGE` | NPU profiling 阶段选择（如 prefill / decode） |
| `SGLANG_PROFILE_WITH_STACK` | 带 Python 调用栈的 profiling，开销较大 |

**检查方式**：
1. 判断当前变更文件是否属于 `best_practice/` 或 `model-tutorials/` 目录
2. 在该文件的脚本代码块中搜索上述三个环境变量名（`export` 语句或行内赋值形式 `VAR=val`）
3. 发现任一项即标记为违规

**修复指引**：移除脚本中的调试环境变量。如果需要在文档中说明如何开启调试，应在正文中另起独立段落描述（如 `<Tip>` 或 `<Warning>` 块），而非混入部署脚本。

---

## M-F-10：用例内信息一致性

**检查范围**：`best_practice/` 目录下 `.mdx` 文件中，`## Optimal Configuration` 之后的每个 `###` 级用例 section。

**规则**：同一用例内，以下五组信息必须一致。任一不一致即报告 🟡 ISSUE。

### 1. 标题 ↔ 元数据

用例标题格式为 `### <Model> <Quant> <NodeDesc> <DatasetDesc> <TPOT>`，其各字段须与紧随其后的 `**field**` 元数据块完全一致：

| 标题中的字段 | 对应的元数据字段 | 示例（标题 → 元数据） |
|---|---|---|
| `<Model>` | `**Model**` | `Qwen3-30B-A3B` → `Qwen3-30B-A3B` |
| `<Quant>` | `**Quantization**` | `W8A8` → `W8A8 INT8`（元数据含更详细描述，但核心量化简称须匹配） |
| `<NodeDesc>` | `**Cards**` + `**Deploy Mode**` | `1P` → Cards=`1`, Deploy Mode=`PD Mixed` |
| `<DatasetDesc>` | `**Dataset**` | `IN3K5 OUT1K5` → `3.5K+1.5K`（格式不同但数值须对应） |
| `<TPOT>` | `**TPOT**` | `37ms` → `37ms` |

标题和元数据中 `<NodeDesc>` 与 `Deploy Mode` 的对应关系：
- `NP` → PD Mixed（N 卡 PD 分离混合部署）
- `NP` + `MD`（含 `D` 标记）→ PD Disaggregation（N 个 prefill 节点 + M 个 decode 节点）

### 2. 标题 ↔ 速查表行

速查表（`### Low Latency` 或 `### High Throughput` 下的 8 列表格）中必须存在一行，其 `Configuration` 列链接的锚点目标（如 `#qwen3-30b-a3b-w8a8-1p-in3k5-out1k5-37ms`）指向该用例标题，且该行的 `Model`、`Hardware`、`Cards`、`Deploy Mode`、`Dataset`、`TPOT`、`Quantization` 列值与元数据一致。

### 3. 速查表 Configuration 链接 ↔ 标题锚点

`Configuration` 列链接格式为 `[Optimal Configuration](#<slug>)`，其中 `<slug>` 必须能解析到文件中某个 `###` 标题。slug 生成规则：标题全小写，空格和 `.` 替换为 `-`，示例：

| 标题 | slug |
|---|---|
| `Qwen3-30B-A3B W8A8 1P IN3K5 OUT1K5 37ms` | `qwen3-30b-a3b-w8a8-1p-in3k5-out1k5-37ms` |
| `Qwen3-30B-A3B BF16 1P IN1K OUT100` | `qwen3-30b-a3b-bf16-1p-in1k-out100` |
| `DeepSeek-R1 W8A8 8x1P IN6K OUT1K6 8.66ms` | `deepseek-r1-w8a8-8x1p-in6k-out1k6-8-66ms` |

如果表中存在 Configuration 链接但文件内找不到对应标题，报告 ISSUE。

### 4. 标题 Dataset 描述符 ↔ Benchmark 命令的 input/output-len

标题中的 `<DatasetDesc>` 格式为 `IN<X>K OUT<Y>K`（如 `IN3K5 OUT1K5` = 输入 ~3500 tokens，输出 ~1500 tokens），须与 Benchmark 命令中的 `--random-input-len` 和 `--random-output-len` 数值对应。允许因 tokenizer 对齐有小幅偏差（≤10% 容差，且不得跨数量级），但格式意义上的值（如 `3.5K` = 3500）必须在同一数量级：

| Dataset 描述符 | 预期 --random-input-len | 预期 --random-output-len |
|---|---|---|
| `IN3K5 OUT1K5` | ~3500 | ~1500 |
| `IN128K OUT1K` | ~131072 | ~1024 |
| `IN1K OUT100` | ~1000 | ~100 |

### 5. 元数据 Dataset ↔ Benchmark 命令

元数据 `**Dataset**` 字段（如 `3.5K+1.5K`）的数值应与 `--random-input-len` / `--random-output-len` 对应，校验方式同检查 4。

### 检查方式

对每个用例 section 依次执行：
1. 解析标题，提取 `<Model>`、`<Quant>`、`<NodeDesc>`、`<DatasetDesc>`、`<TPOT>`
2. 解析元数据块，逐字段对比标题中的信息
3. 在 Low Latency / High Throughput 两个表格中搜索 Configuration 链接指向该 heading slug 的行，校验行列值一致
4. 提取 `###` 标题文本生成 slug，与表格中 Configuration 链接逐一比对，标记未找到目标的死链
5. 解析 Benchmark 代码块中 `--random-input-len` 和 `--random-output-len`，与标题 DatasetDesc、元数据 Dataset 做数值级校验

### 修复指引

- **标题与元数据不一致**：以实际配置为准，统一两处的值（通常以标题中的缩写为准，元数据补全详细描述）
- **表格行缺失或值不匹配**：在对应表中添加或修正该行，确保所有列值与元数据一致
- **Configuration 链接死链**：修正链接中的 slug 以匹配实际标题
- **Benchmark 参数与 Dataset 不一致**：修正 `--random-input-len` / `--random-output-len` 以匹配 Dataset 描述符

---

## M-F-11：新增文档须有完整 YAML 前置元数据

**检查范围**：`ascend-npus/` 目录下所有新增的 `.mdx` 文件。

**规则**：新增 `.mdx` 文件开头必须有 YAML 前置元数据（frontmatter），且至少包含以下必填字段：

```yaml
---
title: "Page Title"
metatags:
  description: "Brief description for SEO and navigation."
---
```

**检查项**：

| 检查项 | 要求 |
|---|---|
| frontmatter 存在 | 文件必须用 `---` 行开头，且有配对的闭合 `---` 行 |
| `title` 必填 | frontmatter 中必须包含 `title` 字段，值须用双引号包裹 |
| `description` 必填 | 必须有 `metatags.description`（优先，与 best_practice / model-tutorials 保持一致）或顶层 `description` |
| `description` 有意义 | 不能是空字符串，不能是象征性单字（如 `"desc"`、`""`），须简要说明页面内容 |

各子目录的 description 参考格式：

| 目录 | 建议格式 | 示例 |
|---|---|---|
| `best_practice/` | `"Best Practice for <ModelName> on Ascend NPU"` | `"Best Practice for Qwen3-8B on Ascend NPU"` |
| `model-tutorials/` | `"Deploy <ModelName> model with SGLang on Ascend NPUs, including ..."` | `"Deploy DeepSeek-R1 model with SGLang on Ascend NPUs, including single-node and multi-node PD disaggregation modes."` |
| 顶层页面（quickstart、FAQ 等） | 简要概括页面核心内容 | `"Quickstart for running SGLang on Ascend NPUs with the official container image, including server launch and test request examples."` |

**检查方式**：
1. 从 PR diff 中识别文件状态为 `added` 的 `.mdx` 文件
2. 读取文件前 10 行，确认以 `---` 开头且存在配对闭合的 `---`
3. 解析 frontmatter 中是否存在 `title`（允许 YAML 多行字符串但禁止缺失）
4. 检查 `metatags.description` 或顶层 `description` 是否存在且非空

**修复指引**：参考同目录下已有文件的 frontmatter 格式，补全缺失字段。`model-tutorials/` 和 `best_practice/` 下文件可参照同目录内任意文件复制并修改 `title` 和 `description`。

---

## M-F-12：成对标点符号须正确闭合

**检查范围**：所有变更 `.mdx` 文件中，diff 的新增或修改行的正文内容（代码块内内容除外）。

**规则**：以下成对出现的符号在正文中必须正确闭合，漏闭合会导致 Markdown/MDX 渲染异常或用户阅读困惑。

### 检查项

| 类别 | 符号 | 示例（合规 / 不合规） |
|---|---|---|
| 粗体 | `**...**` | `**bold text**` ✓ / `**bold text` ✗ |
| 斜体 | `*...*` | `*italic*` ✓ / `*italic` ✗（注：不消费已被 `**` 匹配的 `*`） |
| 删除线 | `~~...~~` | `~~strikethrough~~` ✓ / `~~strikethrough` ✗ |
| 内联代码 | `` `...` `` | `` `code` `` ✓ / `` `code `` ✗ |
| 代码块 | ```` ``` ```` | 每对三反引号须完整开闭 |
| 链接 | `[...](...)` | 方括号和圆括号须各自配对；允许嵌套，如 `[text ![img](url)](link)` |
| 图片 | `![...](...)` | 与链接同理，`!` 后可跟 `[...]` |
| 双引号 | `"..."` | `"hello"` ✓ / `"hello` ✗（排除 YAML frontmatter 中的语法引号） |
| 单引号 | `'...'` | `'example'` ✓ / `'example` ✗（排除英文缩写撇号如 `don't`、`it's`） |
| 中文方括号 | `【...】` | `【注意】` ✓ / `【注意` ✗ |

### 排除范围

| 排除场景 | 原因 |
|---|---|
| 代码块（`` ```...``` ``）内内容 | 代码块中符号有语法含义，不适用此规则 |
| YAML frontmatter（`---...---` 之间） | YAML 语法的引号由 M-F-11 单独检查 |
| 英文缩写撇号（`don't`、`it's`、`won't` 等） | 单引号在此场景是合法语法，不视为未闭合 |
| MDX 组件标签（`<Tabs>`、`</Tabs>` 等） | JSX 标签闭合由 M-F-10 或 A-1 相关逻辑覆盖 |

### 检查方式

1. 从 PR diff 中提取每个变更文件的**新增行**（`+` 前缀）和**修改行**的正文部分
2. 排除代码块（`` ```...``` ``）内行和 YAML frontmatter 行
3. 对同一段落/逻辑块（以空行为界）统计各符号的累计开闭次数：
   - `**` 计数：开闭须成对（偶数次总出现）
   - `*` 计数：排除已被 `**` 消费的 `*` 后，`*` 也须成对
   - `~~` 计数：开闭须成对
   - `` ` `` 计数：单反引号出现总次数须为偶数（每两个一组形成内联代码），排除三反引号已消费的
   - `【` / `】` 计数：开闭次数须相等
   - `"` 计数：双引号在正文中出现总次数须为偶数
   - `'` 计数：单引号在正文中出现总次数须为偶数（排除已知缩写后）
4. 符号计数不等的段落标记为 🟡 ISSUE
5. 允许跨行配对（如 `**` 跨两行），但跨段落配对（中间有空行）标记为 ISSUE
6. 对于 `[]()` 链接，检查每对 `[` 后是否有配对的 `]` 和 `(`...`)`

### 修复指引

- **漏闭合**：在对应位置补上闭合符号（如 `**text` → `**text**`）
- **多闭合**：删除多余的符号或确认是否为内容的一部分（后者需转义 `\*`）
- **跨段落配对**：将段落合并或分别在每个段落内独立闭合

---

## M-F-13：单位使用须符合 SI 规范

**检查范围**：所有变更 `.mdx` 文件中，diff 的新增或修改行里的正文和代码注释（代码块内命令/脚本主体除外）。

**规则**：文档中涉及物理量和单位的表述须符合国际单位制（SI）规范。

### 检查项

#### 1. 数字与单位符号之间须有空格

数值与单位符号之间须有一个半角空格。

| 合规 | 不合规 | 说明 |
|------|--------|------|
| `10 ms` | `10ms` | 数值与单位间缺空格 |
| `100 GB` | `100GB` | 同上 |
| `3.5 GHz` | `3.5GHz` | 同上 |
| `64 °C` | `64°C` | 可用空格，但 `°C` 不强制要求 |

**例外**：角度单位（`°`、`′`、`″`）和百分比（`%`）与数字之间不加空格：`90°`、`50%` 合规。

#### 2. 单位符号大小写须正确

单位符号的大小写由国际标准规定，不可随意更改。

| 合规 | 不合规 | 说明 |
|------|--------|------|
| `ms`（毫秒） | `MS`、`Ms`、`mS` | 毫秒符号全小写；`MS` 是兆西门子 |
| `s`（秒） | `S`、`Sec`、`sec` | 秒符号小写 |
| `GB`（吉字节） | `gb`、`Gb` | 吉字节：G 大写 B 大写；`Gb` 是吉比特 |
| `MB`（兆字节） | `mb` | 兆字节：M 大写 B 大写 |
| `kB`（千字节） | `KB`、`kb` | k 须小写（k = kilo），`K` 是开尔文温度 |
| `GHz` | `ghz`、`Ghz`、`GHZ` | G 大写，Hz 首字母大写 |
| `MHz` | `mhz`、`Mhz` | M 大写，Hz 首字母大写 |
| `W`（瓦特） | `w` | 瓦特符号大写 |
| `V`（伏特） | `v` | 伏特符号大写 |
| `A`（安培） | `a` | 安培符号大写 |

**常见容易混淆的对照**：

| 场景 | 正确 | 错误 | 原因 |
|------|------|------|------|
| 吞吐量单位 | `tok/s` 或 `tokens/s` | `TOK/S`、`Tok/s` | 单位符号区分大小写，`tok` 非标准单位 |
| 带宽/速率 | `GB/s` | `GBps`、`gb/s` | 斜线表示"每"；`GBps` 中 `ps` 是皮秒 |
| 内存容量 | `GB` / `GiB` | `gb`、`G` | B 不可省略（不区分 byte 与 bit 时 `GB` 可接受） |
| 时延 | `ms`、`μs` | `msec`、`usec` | 须用标准符号，`us` 是微秒的非标准缩写 |
| 频率 | `GHz` | `ghz`、`GHZ` | Hz 取自人名专有名词，首字母须大写 |

#### 3. 须使用标准 SI 前缀

| 前缀 | 符号 | 量级 | 合规示例 | 不合规示例 |
|------|------|------|----------|------------|
| 千 | k | 10³ | `kHz`、`kB` | `KHz`（K 大写是开尔文） |
| 兆 | M | 10⁶ | `MHz`、`MB` | `mhz` |
| 吉 | G | 10⁹ | `GHz`、`GB` | `ghz` |
| 太 | T | 10¹² | `TB` | `tb` |
| 毫 | m | 10⁻³ | `ms`、`mm` | `MS` |
| 微 | μ | 10⁻⁶ | `μs` | `us`（`us` 在非技术文档可接受，但推荐 `μs`） |
| 纳 | n | 10⁻⁹ | `ns` | `nsec` |

#### 4. 不得使用已废弃的单位表示法

| 废弃写法 | 应改为 | 说明 |
|----------|--------|------|
| `msec` | `ms` | 毫秒 |
| `usec` | `μs` | 微秒 |
| `sec` | `s` | 秒 |
| `bps`（bits per second） | `bit/s` | 比特每秒 |
| `K`（表示千，如 `K tokens`） | `k`（如 `k tokens`） | 虽然非正式单位语境中 `K` 作为千的简写已被广泛接受，但推荐统一用 `k` |

### 排除范围

| 排除场景 | 原因 |
|----------|------|
| 代码块内命令/脚本主体 | 命令中的参数名、标识符可包含大小写混合（如 `--random-input-len 3500`） |
| YAML frontmatter | 标题和 description 中的固定用语 |
| 产品名称 / 品牌名 / 型号 | 如 `Ascend 910B`、`Atlas 800T A2`、`Qwen3-30B-A3B` |
| 文件路径 / URL | 不做单位检查 |
| 专有缩写 | 如 `TPOT`（Time Per Output Token）、`MFU`（Model FLOPs Utilization） |

### 检查方式

1. 从 PR diff 中提取每个 `.mdx` 文件的新增/修改行的正文和代码注释部分
2. 搜索数值+单位组合的模式（正则：`\d+[\s]?[a-zA-Zμ°]+`）
3. 检查数字与单位之间是否有空格（`°`、`%` 除外）
4. 检查单位符号大小写是否符合上表
5. 检查是否使用了已废弃的单位表示法

### 修复指引

- **缺空格**：在数字与单位之间补一个半角空格
- **大小写错误**：按 SI 规范修正单位符号的大小写
- **废弃表示**：替换为标准单位符号
