# kinetic：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`1b671b9eea0db10c1d22c51bf524d496d6c6736c`
- 提交时间：`2026-11-24T12:35:28+01:00`
- 提交主题：Merge pull request #34 from brianporeilly/pr/vulkan-matrix
- 源仓库：https://github.com/Veloresearch/kinetic.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

独立 Rust 引擎，包含 CPU、CUDA、Vulkan 和统一模型块；重点是重复工作复用、投机、exact state 和混合放置。README 对 split-model、重复编辑 benchmark 的限制有明确披露。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 模型块共享执行语义 | 模型层组织为通用 block，CPU/CUDA/Vulkan 提供执行；从 GGUF/KTC 读取格式元数据。 | 支持范围按 family/backend 而异，不因为 parser 能读就保证能算。 | [crates/kinetic-models/src/model.rs](../../../inferences/kinetic/crates/kinetic-models/src/model.rs)、[crates/kinetic-models/src/loader.rs](../../../inferences/kinetic/crates/kinetic-models/src/loader.rs) |
| 专家缓存按大小分池 | 同大小专家共享 pool，显存按各 pool 的专家总字节比例分配。 | 预留设备空间并计 dense、cache、staging；不能只按专家数量均分。 | [crates/kinetic-models/src/cuda/experts/plan.rs](../../../inferences/kinetic/crates/kinetic-models/src/cuda/experts/plan.rs) |
| CPU/GPU/NVMe 分层 | dense 优先在 GPU，专家装不下时 host/file backing；计划记录 host_rate、copy_rate。 | 适用 MoE；低命中率时 PCIe 与主存成为限制。 | [crates/kinetic-models/src/cuda/experts/plan.rs](../../../inferences/kinetic/crates/kinetic-models/src/cuda/experts/plan.rs)、[crates/kinetic-models/src/cuda/experts/cache.rs](../../../inferences/kinetic/crates/kinetic-models/src/cuda/experts/cache.rs) |
| 持久 prefix state | Warm 在内存字节预算和可选磁盘中保存最长共同前缀；recurrent state 只在已保存 mark 截断。 | GRID=256，按成本可扩到 8192；需对齐 batch 边界才能严格复现相关路径。 | [crates/kinetic-models/src/warm/mod.rs](../../../inferences/kinetic/crates/kinetic-models/src/warm/mod.rs)、[crates/kinetic-models/src/warm/disk.rs](../../../inferences/kinetic/crates/kinetic-models/src/warm/disk.rs) |
| 多请求共享权重 pass | Convoy 收集当前各 sequence 下一次工作，凑齐或至多等 5ms 后一人运行共享 pass；Seat 管理加入退出。 | 吞吐优先会增加等待；慢客户端通过 away 释放设备持有权。 | [crates/kinetic-models/src/convoy.rs](../../../inferences/kinetic/crates/kinetic-models/src/convoy.rs) |
| 树形投机与 proposal 组合 | Draft 表达 token/parent/chance，包含 lookup、MTP、DSpark、companion 等来源。 | target 仍须逐路径验证；不是输出可直接复制输入。 | [crates/kinetic-models/src/speculative/draft.rs](../../../inferences/kinetic/crates/kinetic-models/src/speculative/draft.rs)、[crates/kinetic-models/src/speculative/mod.rs](../../../inferences/kinetic/crates/kinetic-models/src/speculative/mod.rs) |
| 策略和采样一致性 | plain 和 verify 共用 Policy，greedy 可不下载全部 logits；采样按必要范围取 scores。 | 消除大词表 D2H 对 LLM 有用，对图像 denoiser 不一定有对应瓶颈。 | [crates/kinetic-models/src/speculative/policy.rs](../../../inferences/kinetic/crates/kinetic-models/src/speculative/policy.rs) |
| CUDA Graph 与按形状 kernel | CUDA backend 提供 capture/replay、tile/flash attention、小行数/大 batch 不同矩阵乘。 | 可用能力和 batch 数值差异需要单独验收。 | [crates/kinetic-cuda/src/graph.rs](../../../inferences/kinetic/crates/kinetic-cuda/src/graph.rs)、[crates/kinetic-cuda/src/attention.rs](../../../inferences/kinetic/crates/kinetic-cuda/src/attention.rs)、[crates/kinetic-cuda/src/gemv.rs](../../../inferences/kinetic/crates/kinetic-cuda/src/gemv.rs) |
| Hadamard basis 的格式契约 | Folds 读取模型声明的旋转/逆变换权重、sign 表、分块与 layout，要求所有声明都有消费方。 | 它是支持特定 checkpoint 的路径，不代表对任意模型自动做无损量化。 | [crates/kinetic-models/src/folds.rs](../../../inferences/kinetic/crates/kinetic-models/src/folds.rs) |
| 已披露的正确性和统计边界 | README 列出 CPU/GPU split 后续请求可不一致、split 投机可能变慢，编辑加速区分首次与重复。 | 这里只记录本地 README 所描述的版本状态，没有认定后续版本已修复。 | [README.md](../../../inferences/kinetic/README.md) |

## 对 ComfyUI 的借鉴

缓存必须有严格的依赖 key 和状态边界；跨请求凑 pass 要有短等待上限；同一权重的多个任务合批不等于随意同时启动多个 GPU forward。

## 下次跟踪

优先跟踪 split-state 一致性修复和 cold/repeated 分项统计；对 exactness 必须指明相对哪一个 backend、量化和 batch 边界。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
