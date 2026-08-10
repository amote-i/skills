---
name: "ascend-docs-updater"
description: "Updates Ascend NPU best practice docs from test cases. Invoke when user wants to update/regenerate docs, sync docs with test cases, or check doc-testcase consistency."
---

# Ascend 文档更新器

本技能通过运行 `generate_docs.py` 脚本更新 Ascend NPU 最佳实践文档，跟踪用例 commit ID，并展示 diff 摘要供用户确认。

所有路径均相对仓库根，工作目录默认为仓库根，无需 `cd`。

## 何时调用

- 用户要求更新或重新生成 Ascend 最佳实践文档
- 用户想要同步文档与最新用例
- 用户想要检查文档与用例是否一致
- 用户提到每周文档更新或文档刷新

## 关键文件路径（均相对仓库根）

- 生成脚本：`.agents/skills/ascend-docs-updater/generate_docs.py`（唯一真相源，路径自动推导）
- 锚点检查：`.agents/skills/ascend-docs-updater/_check_anchors.js`
- 版本追踪：`.agents/skills/ascend-docs-updater/doc_version.json`
- 用例工作树：`work_dirs/ascend-sglang-testcases/`（`work_dirs/` 已被 `.gitignore` 忽略）
  - 用例目录：`work_dirs/ascend-sglang-testcases/test/registered/ascend/performance/`
  - 用例源不绑定特定远端——任何含 `test/registered/ascend/performance/` 的 git 仓库/分支均可 clone 到此目录（性能用例目前随主仓库的文档分支演进，见 `doc_version.json` 追踪的 commit）
- 输出目录：`docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/`
- 教程目录：`docs/docs/hardware-platforms/ascend-npus/model-deployment/tutorials/`
- utils 文件：`python/sglang/test/ascend/e2e/test_npu_performance_utils.py`（提供 `*_MODEL_PATH` 映射）
- 站点导航：`docs/docs.json`

## 工作流程

### 第 0 步：环境探测与首次配置

运行前自动检查以下前提，任一缺失则向用户给出清晰引导后停止（不自动 clone，避免网络/权限问题卡住）：

1. **用例工作树是否存在**：`work_dirs/ascend-sglang-testcases/test/registered/ascend/performance/`
   - 不存在 → 提示用户把含性能用例的仓库/分支 clone 到 `work_dirs/ascend-sglang-testcases/`（仅首次需要）：
     ```
     git clone <含性能用例的远端> work_dirs/ascend-sglang-testcases
     # 若用例在某分支上：git -C work_dirs/ascend-sglang-testcases checkout <分支>
     ```
   - 存在但 `test/registered/ascend/performance/` 为空 → 警告，可能 clone 了错误的仓库/分支。

2. **输出目录是否存在**：`docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/`
   - 不存在 → 报错。说明文档仓库布局与 Skill 预期不符（路径可能再次迁移），需先更新 Skill 里的路径常量。

3. **utils 文件是否存在**：`python/sglang/test/ascend/e2e/test_npu_performance_utils.py`
   - 不存在 → 报错。模型路径映射将缺失。

4. **版本追踪文件**：`.agents/skills/ascend-docs-updater/doc_version.json`
   - 不存在 → 视为首次运行，以用例仓库当前 HEAD 为基线初始化（见第 1 步的基线校验）。

全部通过后进入第 1 步。

### 第 1 步：记录当前用例 commit 并校验基线

读取 `.agents/skills/ascend-docs-updater/doc_version.json` 获取 `last_testcase_commit`，该 commit ID 代表当前文档对应的用例版本。

获取用例仓库当前 HEAD：
```
git -C work_dirs/ascend-sglang-testcases log -1 --format="%H %s"
```

**基线校验**：确认 `last_testcase_commit` 在用例仓库历史中存在：
```
git -C work_dirs/ascend-sglang-testcases cat-file -e <last_testcase_commit> 2>/dev/null
```
- 若失败（commit 不在用例仓库历史中，常见于 Skill 迁移机器后基线失配）：警告用户"基线 commit 失配，本次将以全量重新生成，跳过第 4 步的增量 diff，直接进入第 5 步"。本次成功更新后（第 9 步）会用真实 HEAD 覆盖该字段，完成自愈。
- 若成功但与当前 HEAD 不一致：警告用户用例仓库可能被手动更新但未同步文档，询问是否继续（以 `last_testcase_commit` 为基线）。

### 第 2 步：拉取用例仓库

```
git -C work_dirs/ascend-sglang-testcases pull
```

记录拉取后的新 HEAD commit ID：
```
git -C work_dirs/ascend-sglang-testcases log -1 --format="%H %s"
```

### 第 3 步：拉取文档仓库

文档仓库即当前主仓库。提示用户确保主仓库当前分支已拉取最新：
```
git pull
```
（在仓库根执行；若用户在特性分支上工作，按其正常 git 流程处理，Skill 不强制 remote/分支。）

### 第 4 步：对比用例变更

> 若第 1 步判定基线失配，跳过本步，直接进入第 5 步（全量重生成）。

运行：
```
git -C work_dirs/ascend-sglang-testcases diff <last_testcase_commit>..HEAD --stat -- test/registered/ascend/performance/
```

然后获取详细 diff：
```
git -C work_dirs/ascend-sglang-testcases diff <last_testcase_commit>..HEAD -- test/registered/ascend/performance/
```

如果没有检测到变更，报告"自上次更新以来用例无变更，文档已是最新"并停止。

否则，以结构化形式总结用例变更：
- **新增文件**：新增的用例（新模型配置）
- **删除文件**：删除的用例
- **重命名文件**：文件重命名
- **修改文件**：参数变更，高亮关键差异（环境变量、命令行参数、benchmark 参数、类变更）

### 第 5 步：检查文档仓库工作区

运行：
```
git status --short docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/
```

如果有未提交的 .mdx 文件变更，警告用户并询问是否继续（脚本会覆盖 .mdx 文件）。

### 第 6 步：运行生成脚本

执行（脚本路径相对仓库根，内部自动推导所有路径）：
```
python .agents/skills/ascend-docs-updater/generate_docs.py
```

脚本会在用例目录不存在或 utils 文件缺失时报错退出（见第 0 步）。

### 第 6-2 步：一致性校验报告

脚本运行时会在 stderr 输出「场景/部署/压测三者一致性」校验报告。脚本对每个性能用例执行 C1-C8 规则（定义见 `generate_docs.py` 的 `validate_config_consistency`），不合规的用例在此汇总：

```
⚠️  一致性校验报告（场景/部署/压测三者匹配性，共 N 条警告）
  [模型目录/文件名]
    [Cn] 具体问题描述
```

**处理策略**：警告但不阻断生成——不合规用例仍生成文档，问题暴露给人工复核。Skill 应：
1. 把校验报告完整展示给用户
2. 逐条说明问题性质（C1-C8 含义见下方规则表）
3. 询问用户：是修复用例本身（归用例维护者），还是调整脚本解析逻辑，或暂时忽略

**规则速查**（每条的目的，便于维护者理解与扩展）：

| 规则 | 校验内容 | 目的 |
|---|---|---|
| C1 | 测试基类 ↔ deploy_mode 标签自洽 | 场景（基类）与分类（标签）一致 |
| C2 | 文件名 PD 标记(1p1d/pd_sep) ↔ 基类 | 文件名声明的场景与基类一致 |
| C3 | PD 分离须有 prefill_args + decode_args | 部署结构完整 |
| C4 | PD 分离两端须含 --tp-size / --nnodes | prefill/decode 部署对称 |
| C5 | benchmark 须含延迟(tPot) + 吞吐指标 | 压测结果完整，有最佳实践价值 |
| C6 | 文件名 in/out 长度 ↔ benchmark.input/output_len | 压测参数与文件名声明的场景一致 |
| C7 | num_prompts ≥ max_concurrency | 压测请求数充足 |
| C8 | tp-size 不超过硬件物理上限（A3: 2×cards, A2: cards）| 硬件规模与并行度自洽 |

### 第 7 步：展示文档 diff 并交叉验证

脚本完成后，运行：
```
git diff --stat docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/
```

然后：
```
git diff docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/
```

**交叉验证**：对比用例变更（第 4 步）与文档变更（本步），确认它们匹配：

1. **新增用例 → 新增文档段落**：每个新增的用例文件应在对应 .mdx 中产生新段落
2. **删除用例 → 删除文档段落**：每个删除的用例文件应从 .mdx 中删除对应段落
3. **修改用例 → 更新文档段落**：每个修改的用例文件应更新对应段落
4. **无意外文档变更**：文档变更应仅来自用例变更。如果文档有变更但用例无变更，标记为可疑。

**参数级验证**：对于新增和修改的用例，进一步验证文档中的参数与用例参数一致：

1. 使用 `generate_docs.py` 中的 `extract_config_from_file` 提取用例的环境变量和命令行参数
2. 在对应文档段落中搜索每个环境变量（`export KEY=`）和命令行参数（`--flag`），确认都已出现
3. 对于关键参数值（如 `--tp-size`、`--dp-size`、`--mem-fraction-static`、`--moe-a2a-backend`、`--deepep-mode`、`--speculative-*` 等），对比用例值与文档值是否一致
4. PD 分离模式分别验证 prefill 和 decode 的参数

以汇总表格形式展示交叉验证结果：

| 用例变更 | 文档变更 | 段落匹配 | 参数一致 | 状态 |
|---|---|---|---|---|
| 新增: test_npu_xxx.py | xxx.mdx 新增段落 | ✅ | ✅ | ✅ 匹配 |
| 修改: test_npu_yyy.py (环境变量变更) | yyy.mdx 更新环境变量 | ✅ | ⚠️ --tp-size 用例=8 文档=4 | ⚠️ 参数不一致 |
| 删除: test_npu_zzz.py | zzz.mdx 删除段落 | ✅ | - | ✅ 匹配 |
| (无) | www.mdx 意外变更 | - | - | ⚠️ 需排查 |

### 第 7-2 步：同步 docs.json 导航

best-practices 目录下的 .mdx 增加或删除时，需要同步更新 `docs/docs.json` 中 "Best Practices" group 的 pages 列表。

读取 `docs/docs.json`，递归遍历整个 JSON 树，找到 `"group": "Best Practices"`（注意复数）且含 `"pages"` 数组的 dict，将其 `pages` 数组替换为当前 best-practices 目录下所有 .mdx 文件（不含扩展名）按字母序排列的列表。

路径前缀为 `docs/hardware-platforms/ascend-npus/model-deployment/best-practices/`，即：
```
expected_pages = [
    f"docs/hardware-platforms/ascend-npus/model-deployment/best-practices/{slug}"
    for slug in sorted(mdx_filenames_without_ext)
]
```

实现逻辑：
1. 扫描 `docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/` 下所有 `*.mdx` 文件，排序得到 slug 列表
2. 构造 `expected_pages`（见上式）
3. 加载 `docs/docs.json`，递归遍历整个 JSON 树，找到 `"group": "Best Practices"` 且含 `"pages"` 的 dict，对比 `pages` 与 `expected_pages`
4. 如果不一致，用 `expected_pages` 替换并写回（保持原有 indent 2、ensure_ascii=False）
5. 如果没有找到 "Best Practices" group，不修改 `docs.json`

写回时用 `json.dump(obj, f, indent=2, ensure_ascii=False)` 保持格式兼容。

### 第 7-3 步（可选）：锚点检查

运行锚点校验脚本，检查生成文档中的内部锚点是否都有对应标题：
```
node .agents/skills/ascend-docs-updater/_check_anchors.js
```

报告 broken anchors 供用户判断是否需要修复脚本输出。

### 第 8 步：确认

询问用户确认变更：
- 用户批准：用新 commit ID 和当前日期更新 `doc_version.json`，变更保留在工作区
- 用户拒绝：运行 `git checkout -- docs/docs/hardware-platforms/ascend-npus/model-deployment/best-practices/` 回滚（注意：不会回滚 `docs.json`，需单独 `git checkout -- docs/docs.json`）

### 第 9 步：更新版本追踪

用户确认后，更新 `.agents/skills/ascend-docs-updater/doc_version.json`：
```json
{
  "last_testcase_commit": "<拉取后的新commit_id>",
  "last_update_date": "<今天日期>",
  "last_update_summary": "<变更摘要>"
}
```

对于第 1 步判定基线失配的首次运行，此处即将 `last_testcase_commit` 修正为真实 HEAD，完成基线自愈。

## 注意事项

- 脚本是文档生成的唯一真相源；所有路径由脚本顶部常量基于 `__file__` 自动推导，不要在 SKILL.md 或命令行里覆盖
- best-practices 目录下所有 .mdx 文件都是全量生成的——不要手动编辑，否则会被覆盖
- 如果脚本输出不正确，修复脚本本身，而不是 .mdx 文件
- 每次成功更新文档后必须更新 `doc_version.json` 以维护追溯链
- 运行脚本前务必拉取两个仓库（用例 clone + 文档仓库）以确保使用最新代码
- 用例工作树在 `work_dirs/ascend-sglang-testcases/`（已被 `.gitignore` 忽略），不入库，每位用户首次使用时各自 clone
- **场景/部署/压测三者一致性原则**：只有当用例的「场景」（文件名声明 + 测试基类）、「部署」（envs/args，PD 分离时含 prefill/decode 拆分）、「压测」（benchmark 参数）三者完全匹配时，才认可其作为性能脚本抽取最佳实践。`validate_config_consistency`（C1-C8）把此原则固化为校验规则；维护者新增用例形态（如新的部署模式）时，应同步增补对应规则，并在 SKILL.md 的规则速查表登记
- 性能基类名（`PERF_SINGLE_NODE_BASE` / `PERF_PD_MIX_BASE` / `PERF_PD_SEP_BASE`）若 utils 侧重命名，只需改脚本顶部常量定义处
