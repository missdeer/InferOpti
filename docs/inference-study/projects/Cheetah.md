# Cheetah：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`e3451a52872c95e8c3af0d347c20dbd55004c6fa`
- 提交时间：`2026-11-25T13:18:48+08:00`
- 提交主题：1.2.1
- 源仓库：https://github.com/DA3M0/Cheetah.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

规模较小的 Rust/C 引擎，Safetensors 准备格式、native CPU 后端和独立 Vulkan 后端。适合学习内存页控制的基本机制，也展示了声明的 tier 与真实设备迁移可能不一致。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| mmap 权重准备格式 | manifest 索引 shard、tensor offset、dtype 和 shape，native CPU 直接读映射数据。 | 信任边界要检查 tensor span；格式支持和量化缓存路径不同。 | [src/chat/native/store.rs](../../../inferences/Cheetah/src/chat/native/store.rs) |
| 真实 host 页控制 | native set_tier 为 resident 主动 touch 页，为 disk 用 MADV_DONTNEED；按层释放 disk tier。 | DONTNEED 是映射/RSS 提示，不等于把文件从系统 page cache 清除，也不是硬内存上限。 | [src/chat/native/store.rs](../../../inferences/Cheetah/src/chat/native/store.rs) |
| Ant 驻留计划和预算 | 按 unit 跟踪计划层级、resident/evicted 集合，低价值 unit 优先退让。 | engine 明说 accelerator/host-resident 后端部分转换只更新 bookkeeping；不能认定 Vulkan 已实现同等真实分页。 | [src/chat/engine.rs](../../../inferences/Cheetah/src/chat/engine.rs)、[cheetah_memory.c](../../../inferences/Cheetah/cheetah_memory.c) |
| CPU SIMD 与量化点积 | native toolbox 分派 BF16/F16/F32 与 Q8/Q4 计算，AVX2 等内核避免通用 tensor 框架开销。 | Q4/Q8 是精度选择；不能认为相对原始模型零损失。 | [src/chat/native/kernels.rs](../../../inferences/Cheetah/src/chat/native/kernels.rs)、[src/chat/native/ops.rs](../../../inferences/Cheetah/src/chat/native/ops.rs)、[src/chat/precision/q4.rs](../../../inferences/Cheetah/src/chat/precision/q4.rs) |
| Chunked prefill | native CPU 把 prompt 分块做 batched GEMM，复用权重读取。 | CHEETAH_PREFILL_CHUNK 控制行数，长输入需兼顾 scratch 与延迟。 | [src/chat/native/ops.rs](../../../inferences/Cheetah/src/chat/native/ops.rs)、[src/chat/native/decoder.rs](../../../inferences/Cheetah/src/chat/native/decoder.rs) |
| Q8 KV 和前缀快照 | 可选 Q8 KV 减少常驻与快照容量；PrefixCache 查最长 token 前缀并 LRU 淘汰。 | 默认浮点 KV，CHEETAH_KV_Q8 需显式开启；命中/未命中首轮分开测。 | [src/chat/native/ops.rs](../../../inferences/Cheetah/src/chat/native/ops.rs)、[src/chat/engine.rs](../../../inferences/Cheetah/src/chat/engine.rs) |
| Vulkan 压缩权重读取 | GLSL Q4 shader 在乘加中读取 nibble 与 row scale，省去全量 FP32 展开。 | 当前所读 shader 是逐输出循环 K 的基础实现，没有据此认定 Tensor Core 等级优化。 | [matmul_q4.comp](../../../inferences/Cheetah/matmul_q4.comp)、[src/chat/vulkan/transformer.rs](../../../inferences/Cheetah/src/chat/vulkan/transformer.rs) |
| 后端边界和热启动 | ModelForward 隔离 native/Vulkan；serve 预热后接请求，重用持久引擎。 | native 为主要页控制证据；不同后端的支持状态应分别列出。 | [src/chat/engine.rs](../../../inferences/Cheetah/src/chat/engine.rs)、[src/chat/registry.rs](../../../inferences/Cheetah/src/chat/registry.rs) |

## 对 ComfyUI 的借鉴

适合学习 mmap 生命周期、格式边界与前缀 key；不宜把 Ant 的统一命名直接当作成熟 GPU 分层实现。可用它提醒团队：预算表必须和 driver/RSS 观测核对。

## 下次跟踪

后续重点检查 Vulkan 真正迁移与回收的调用链，区分 RSS、OS cache、device allocation；记录 native CPU 的数据规模。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
