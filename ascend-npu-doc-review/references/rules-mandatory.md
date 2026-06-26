# 重要级别规则（Mandatory）

必须全部通过，不通过则阻止合入。分为两类：

| 类别 | 编号 | 说明 | 违反后果 | 严重级别 |
|------|------|------|----------|----------|
| **正确性** | M-C-x | 影响脚本/命令的运行时正确性 | 用户按文档操作会直接报错 | 🔴 BLOCK |
| **重要格式** | M-F-x | 影响文档规范性和可维护性 | 用户可能困惑，但脚本能跑通 | 🟡 ISSUE |

> 两类规则均为"必须修复才可合入"。区别仅在输出报告的严重级别标记：M-C-x → 🔴 BLOCK，M-F-x → 🟡 ISSUE（与 SKILL.md 的严重级别映射保持一致）。

**规则总数**：正确性 1 条，重要格式 8 条。

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
