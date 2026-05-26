# 构建系统检视检查清单

适用于 `docker/`、`python/pyproject.toml`、`sgl-model-gateway/Cargo.toml` 及所有构建配置变更。

> **作用域限制：** 本清单仅覆盖 `docker/`、`python/pyproject.toml`、`sgl-model-gateway/Cargo.toml`。其他构建文件（如 `sgl-kernel/` 的 CMakeLists、`rust/` 的 Cargo 配置）不在检视范围内。
> **最后验证：** 2026-05

---

## 1. Dockerfile（`docker/`）

### 语法与结构
- [ ] Dockerfile 语法正确，可成功构建（无遗漏的多阶段构建依赖）
- [ ] 构建步骤合理：先复制依赖文件再复制源码，充分利用 Docker 层缓存
- [ ] 多阶段构建中，各阶段目标明确（builder vs runtime）

### 基础镜像
- [ ] 基础镜像版本明确（使用具体 tag，不用 `latest`）
- [ ] 基础镜像来源可信（官方镜像或已审批的内部镜像）
- [ ] 基础镜像中的系统包版本与项目需求兼容

### 构建步骤
- [ ] 无不必要的 `apt-get` 安装或已最小化
- [ ] Python 依赖安装使用 `--no-cache-dir` 避免缓存残留
- [ ] 敏感信息未硬编码在 Dockerfile 中（密钥、token、内网地址）
- [ ] EXPOSE 端口与实际服务端口一致
- [ ] ENTRYPOINT/CMD 正确，参数化合理

### 镜像大小优化
- [ ] 无构建工具残留（编译器、开发头文件等不应出现在 runtime 镜像中）
- [ ] 清理了包管理器缓存（`apt-get clean`、`rm -rf /var/lib/apt/lists/*`）
- [ ] 未复制不必要的文件（使用 `.dockerignore` 过滤）

## 2. Python 依赖（`python/pyproject.toml`）

### 依赖版本管理
- [ ] 新增依赖指定了版本下界（`>=x.y.z`），不用无约束引用
- [ ] 无重复依赖（同一包出现多次，可能版本冲突）
- [ ] 核心依赖与开发/测试依赖分组正确（`dependencies` vs `optional-dependencies`）
- [ ] 移除的依赖确认为不再使用（全局搜索无残留 import）

### 兼容性
- [ ] 新增依赖的 Python 版本要求与项目最低支持版本兼容
- [ ] 新增依赖与现有依赖无已知冲突（特别是 PyTorch/CUDA 版本矩阵）
- [ ] 版本上界约束合理，不过度限制（避免 `==x.y.z` 精确锁定，除非有充分理由）

### 元数据
- [ ] 项目版本号符合语义化版本规范
- [ ] `requires-python` 与实际最低版本一致
- [ ] entry_points / scripts 配置正确

### 打包配置

- [ ] `entry_points` 的 `console_scripts` 指向的入口函数实际存在且可正确导入
- [ ] `project.dependencies` 与 `project.optional-dependencies` 分组正确、无冲突
- [ ] 不打包无关文件（通过 `tool.setuptools.packages` 或 `MANIFEST.in` 精确控制）

## 3. Rust 依赖（`sgl-model-gateway/Cargo.toml`）

### 依赖版本管理
- [ ] 新增依赖指定了最低版本（`x.y.z`），不用无约束的 `*`
- [ ] 无重复依赖（同一 crate 在 `dependencies` 和 `dev-dependencies` 中版本不一致时需确认）
- [ ] `features` 配置合理：仅启用需要的 feature，不盲目 `full`
- [ ] 移除的依赖确认为不再使用

### 兼容性与安全
- [ ] 新增 crate 为社区主流选择（非冷门/废弃项目）
- [ ] 无已知安全漏洞（可通过 `cargo audit` 检查）
- [ ] 依赖版本与 MSRV（最低支持 Rust 版本）兼容

### Feature Flags
- [ ] 自定义 feature 命名清晰，用途明确
- [ ] Feature 间的互斥/依赖关系正确
- [ ] 默认 feature 集合最小化（`default` 只包含核心功能）

## 4. 通用构建检查

### 可复现性
- [ ] 构建过程可复现（固定版本号、锁定文件存在且同步）
- [ ] 无隐式依赖本地环境（硬编码路径、环境特定配置）
- [ ] CI 构建与本地构建结果一致

### 跨平台
- [ ] Docker 构建支持目标平台（x86/ARM）无遗漏
- [ ] 无硬编码的平台特定路径（`/usr/local/cuda` 应参数化）
- [ ] NPU 适配相关构建参数（CANN 版本、Ascend 工具链路径）正确配置

> **严重等级定义及映射见 SKILL.md 的"严重等级定义"节和"跨维度严重等级映射"节。**
