# 代码检视检查清单

适用于 `python/`（含 `python/sglang/` 全部子目录）及 `sgl-model-gateway/` 下的所有源码变更。

> **作用域限制：** 本清单仅覆盖 `python/` 和 `sgl-model-gateway/` 目录。`sgl-kernel/`、`rust/`、`proto/` 等不在检视范围内。
> **最后验证：** 2026-05（与 `python/sglang/srt/utils/`、`python/sglang/srt/arg_groups/` 等目录结构同步校验）

---

## 1. 正确性

- [ ] 逻辑与设计意图一致（检查上下文，不能只看 diff）
- [ ] 边界情况已处理：空输入、`None` 值、零长度 tensor、超大 batch
- [ ] 差一错误：循环边界、切片索引、序列长度
- [ ] 错误路径：异常被正确捕获和传播，未被静默吞掉
- [ ] 返回值：tensor 操作的 shape、dtype、device 正确
- [ ] 条件分支：所有分支可达且正确（尤其 `if/elif/else` 链）
- [ ] 数值正确性：类型转换顺序、累加精度（float32 vs float16/bfloat16）

## 2. SGLang 运行时专项（`python/sglang/srt/`）

### 模型实现（`srt/models/`）
- [ ] 新模型类已在 `python/sglang/srt/models/registry.py` 中通过 `ModelRegistry.register` 注册
- [ ] 权重加载器正确映射 HuggingFace 权重名到 SGLang 参数名
- [ ] Forward pass 正确处理 `forward_batch`（prefill vs decode、extend vs token）
- [ ] MoE 模型：专家路由逻辑、负载均衡、专家并行兼容性
- [ ] 注意力：正确使用 `ForwardBatch` 注意力元数据（`extend_seq_lens`、`seq_lens`、`prefix_lens`）
- [ ] 多模态模型：pixel values / image embeddings 正确处理和交织

### 服务器与入口（`srt/entrypoints/`）
- [ ] 新 server 参数在 `server_args.py` 中定义，包含类型、默认值和帮助文本
- [ ] 新 API 端点遵循 OpenAI 兼容的响应 schema
- [ ] 请求校验：提前拒绝无效输入并给出清晰错误信息
- [ ] 异步正确性：async handler 中无阻塞调用，正确使用 `asyncio`

### 调度器与内存（`srt/managers/`、`srt/mem_cache/`）
- [ ] 内存管理：不再需要的 tensor 已释放，无引用泄漏
- [ ] KV cache 操作：page/table 操作正确，token 计数无差一错误
- [ ] 调度决策：prefill/decode 优先级正确，树结构完整性得以维护
- [ ] Batch 操作：正确处理 batch 内变长序列

### 算子层（`srt/layers/`）
- [ ] 量化：scale/zero-point 处理正确，shape 校验完整
- [ ] 注意力后端：mask 处理正确，prefill 和 decode 均适用
- [ ] LoRA：权重合并/分离正确，adapter 之间无交叉污染
- [ ] 张量并行：split/merge 操作正确，all-reduce 正确性

### 分布式（`srt/distributed/`）
- [ ] 集合通信：所有 rank 到达相同的集合通信调用（无死锁）
- [ ] 张量并行：权重正确分区，通信匹配
- [ ] 流水线并行：stage 边界正确，micro-batch 处理正确
- [ ] 专家并行：dispatch 和 combine 逻辑、负载均衡

### 推测解码（`srt/speculative/`）
- [ ] 命名遵循 `reference/naming-rules.md` 第 1 节规则
- [ ] Draft/verify 流程：token 计数核算正确（bonus token 语义）
- [ ] EAGLE/ngram：树结构维护正确，拒绝采样正确

### 编译与计算图（`srt/compilation/`）
- [ ] 计算图捕获：静态 shape，捕获区域内无动态控制流
- [ ] `torch.compile` 兼容性：无不受支持的操作，graph break 位置正确
- [ ] 分段计算图：边界条件已处理

### 分离式推理（`srt/disaggregation/`）
- [ ] Prefill/decode 通信：token 传输正确，传输边界无数据丢失
- [ ] KV cache 传输：decode 开始前完成完整传输
- [ ] 故障处理：一侧崩溃时的行为是否合理

### 语言层与 CLI（`srt/lang/`、`srt/cli/`）
- [ ] 语言特性（grammar backends、tool calling 等）的注册和调度逻辑正确
- [ ] CLI 入口点与 `ServerArgs` 参数名一致，无失效的入口点引用

## 3. Model Gateway（`sgl-model-gateway/`）

### 路由与负载均衡
- [ ] 路由策略：请求分发逻辑正确，权重/优先级配置合理
- [ ] 负载均衡：worker 选择算法正确，无热点或饥饿
- [ ] PD 路由：prefill/decode 路由到正确的 worker 集合

### 可靠性
- [ ] 重试/超时：上游故障时正确重试，不会无限阻塞
- [ ] 熔断器：阈值配置合理，状态转换正确
- [ ] 限流：速率限制逻辑正确，不会误杀合法请求

### 安全与认证
- [ ] mTLS：证书验证逻辑正确
- [ ] Auth 中间件：API key / token 校验正确
- [ ] 不记录敏感 header（Authorization 等）

### Rust 代码质量
- [ ] 错误处理：使用 `Result` / `anyhow` / `thiserror`，不 `unwrap()` 生产路径
- [ ] 异步正确性：无 `.await` 在同步上下文、无阻塞调用在 async 函数中
- [ ] 生命周期与所有权：无不必要的 `clone()`，引用生命周期正确
- [ ] 并发安全：`Send`/`Sync` 约束正确，锁粒度合理，无死锁风险

## 4. JIT 内核（`python/sglang/jit_kernel/`）

- [ ] block size / num_warps / num_stages 适合目标硬件
- [ ] 边界检查：最后一个 block 的越界访问已 mask
- [ ] 原子操作：内存序正确，无竞态条件
- [ ] 指针运算：stride、offset 和元素大小正确
- [ ] dtype 处理：fp16、bf16、fp32、fp8 均正确处理（如适用）

## 5. 性能

- [ ] 无不必要的 `torch.cuda.synchronize()` 调用（主要延迟杀手）
- [ ] 热路径中无不必要的 `.cpu()` / `.numpy()` 传输
- [ ] 无隐式 GPU-CPU 同步点：`.item()`、`print(tensor)`、基于 tensor 值的条件判断
- [ ] Tensor 操作在安全处使用原地操作（避免不必要的内存分配）
- [ ] 批处理：操作尽量批量执行而非逐项循环
- [ ] 内存：大型中间 tensor 及时释放（无不必要的持有）

## 6. 安全

- [ ] 无硬编码密钥、API key 或 token
- [ ] 不从环境变量或配置文件读取未经校验的密钥/路径直接传递给高权限操作
- [ ] 无对不可信输入的 `eval()` 或 `exec()`
- [ ] 文件路径：用户提供的路径无路径遍历漏洞
- [ ] 输入校验：server endpoint 校验请求 payload
- [ ] 不记录敏感数据（debug 日志中的用户 prompt、模型权重）

## 7. 调试残留

- [ ] 无遗留的 `breakpoint()` 或 `import pdb` 调用
- [ ] 无调试用 `print()` 语句（生产路径中的日志应使用 `logging`/`logger`）
- [ ] 无 `TODO`/`FIXME`/`HACK`/`XXX` 注释未关联 issue 追踪编号
- [ ] 无被注释掉的代码块（pre-commit hook 不覆盖此类检查）

## 8. 兼容性

- [ ] 向后兼容：现有 API 签名未被破坏（keyword-only 参数、默认值）
- [ ] OpenAI API 兼容：受影响端点的响应格式符合规范
- [ ] 多后端兼容：CUDA 特定代码有适当守卫以兼容 ROCm/NPU（如 `if is_cuda(): ... else: ...`）。对 NPU 适配尤为重要：新增 CUDA 代码路径需用 `is_npu()`（定义在 `python/sglang/srt/utils/common.py`）正确分流，避免 NPU 运行时误入 CUDA 专用路径导致崩溃
- [ ] Protocol Buffers：消息变更向后兼容（新增 optional 字段，而非重命名/删除）。**注意：** `proto/` 不在检视范围内，此条仅适用于 `python/` 中直接引用 protobuf 的代码
- [ ] Python 版本：未使用超出最低支持版本的语言特性
- [ ] 导入路径：无因移动/重命名模块导致的外部导入破坏

## 9. 代码结构

- [ ] 死代码：无注释掉的代码、不可达分支或未使用的参数
- [ ] 导入卫生：无循环导入，重量级可选依赖使用延迟导入

> **严重等级定义及映射见 SKILL.md 的"严重等级定义"节和"跨维度严重等级映射"节。**
