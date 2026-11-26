# sync-strata：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`22b5db9ddd60768fab3a33cb19fbde36084e2dec`
- 提交时间：`2026-11-21T23:36:23+09:00`
- 提交主题：Point engine/ to sync-strata-engine, a fork of Strata that keeps upstream's history and authors; add CREDITS.md
- 源仓库：https://github.com/hiruhiru2005/sync-strata.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。
- **engine 子模块 commit：`824dc0a7811456465bb9864c676381c2d7c72b5f`**（外层应用与内核必须同时定位）。

外层是 Sync-Strata 应用，engine/ 是独立 Git 子模块。优化重点是 Sync 量化包与 trellis 格式/内核协同；外层 commit 不能代替 engine commit。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 量化包和执行器联合设计 | Sync-XS/S/M 使用与 upstream 不同的模型/engine 格式，配套 CPU/GPU 读取和计算。 | README 明示与 upstream Strata、llama.cpp 格式不兼容。 | [README.md](../../../inferences/sync-strata/README.md)、[engine/src/kernels/cpu/tq_cpu.cpp](../../../inferences/sync-strata/engine/src/kernels/cpu/tq_cpu.cpp) |
| 可替换 trellis codebook | tq_lut 装载内置或 pack 指定 LUT，校验内容并生成 host/device 所需形式。 | codebook 和 bitstream 必须成对；这不是普通均匀 INT2 的替换。 | [engine/src/kernels/tq_lut.cpp](../../../inferences/sync-strata/engine/src/kernels/tq_lut.cpp) |
| codebook 的 int8 表示 | 将二维 codebook 点映射为 int8 pair 和全局 scale，供整型点积路径使用。 | LUT 量化本身有误差；不能标为对原始浮点模型无损。 | [engine/src/kernels/cuda/tq_lut_i8.cuh](../../../inferences/sync-strata/engine/src/kernels/cuda/tq_lut_i8.cuh) |
| 按专家/投影混合 trellis 率 | tq_mix 允许同层不同专家的 gate/up 和 down 使用不同 K，随 blob 传递对应信息。 | K 是格式参数，不能一律当成每权重 K bit；接口注明 QuantLab/local builds 范围。 | [engine/include/strata/kernels/tq_mix.hpp](../../../inferences/sync-strata/engine/include/strata/kernels/tq_mix.hpp) |
| CPU ISA 分派 | TQ CPU 路径识别 AVX2、AVX-VNNI、AVX512 等，实现表解码/计算路径。 | 先检查 ISA 再选择；同一最优路径不保证跨 CPU 成立。 | [engine/src/kernels/cpu/tq_cpu.cpp](../../../inferences/sync-strata/engine/src/kernels/cpu/tq_cpu.cpp)、[engine/src/kernels/cpu/tq_avx2.cpp](../../../inferences/sync-strata/engine/src/kernels/cpu/tq_avx2.cpp)、[engine/src/kernels/cpu/tq_avx512.cpp](../../../inferences/sync-strata/engine/src/kernels/cpu/tq_avx512.cpp) |
| 激活解包复用实验 | UNPACK_ONCE 让池任务复用已解包 q8/scale/sum，避免每个任务重复准备。 | 源码注明实验/特定精确行路径；并非普遍默认。 | [engine/src/kernels/cpu/tq_cpu.cpp](../../../inferences/sync-strata/engine/src/kernels/cpu/tq_cpu.cpp) |
| GPU 量化点积和 MMQ | IQ/TQ GPU kernel 与 prompt MMQ 配套，压缩权重在运算时解码，减少展开存储。 | 额外 lookup/解码是否划算取决于矩阵形状和批大小。 | [engine/src/kernels/cuda/iq_kernels.cu](../../../inferences/sync-strata/engine/src/kernels/cuda/iq_kernels.cu)、[engine/src/prefill/moe_mmq.cu](../../../inferences/sync-strata/engine/src/prefill/moe_mmq.cu) |
| 继承分层、图与投机 | 保留 Strata 的专家缓存、CPU 池、CUDA Graph、prefill/decode 分路及 drafter。 | 要用 engine 的具体版本对照；外层 UI/安装器本身不提供新的推理数学。 | [engine/src/core/expert_cache.cpp](../../../inferences/sync-strata/engine/src/core/expert_cache.cpp)、[engine/src/core/graph.cpp](../../../inferences/sync-strata/engine/src/core/graph.cpp)、[engine/src/spec/draft_policy.cpp](../../../inferences/sync-strata/engine/src/spec/draft_policy.cpp) |

## 对 ComfyUI 的借鉴

借鉴“先为目标硬件确定压缩格式，再把加载、传输、激活准备和 kernel 一起设计”。对 DiT 需重新校准量化质量和大 GEMM 吞吐；不能把低位宽 LLM pack 直接当作图像质量结论。

## 下次跟踪

单独跟踪外层与 engine；区分发布 Sync 格式和 QuantLab 的 local-only 实验；跟踪 AVX2 无 AVX512 的真实 CPU 成本。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
