# basalt：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`5b8a82c6b7e1069adbce42854e0732b2b19e4f32`
- 提交时间：`2026-11-25T13:44:41-07:00`
- 提交主题：community: miloaissatu's RTX 5080 runs on 0.5.7 (Inkling-2-Mini int8 113.8, Flash-Next IQ3_S 135.6 tok/s decode) (#749)
- 源仓库：https://github.com/jesdga95/basalt.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

Strata 派生、面向 Blackwell 的专用实现。其价值是目标硬件明确后，把格式、持久 kernel、投机与专家 placement 做到一起；不能用 sm_120 专用收益推断 RTX 30/40 的收益。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| Blackwell 专用格式 | NVFP4 planes、相应 grouped/window kernel 与 pack 转换配套，另保留 dense Q6_K 等格式。 | 具体格式具体 kernel；不能把所有低位宽路径都称为原生 FP4 Tensor Core。 | [src/kernels/cuda/nvfp4_planes.cu](../../../inferences/basalt/src/kernels/cuda/nvfp4_planes.cu)、[src/kernels/cuda/window_gemm_nvfp4.cu](../../../inferences/basalt/src/kernels/cuda/window_gemm_nvfp4.cu) |
| 小验证窗口专用 kernel | NVFP4 window 按 1/2/4/8 rows 选择实例；有 SwiGLU 成对变体减少重复权重流。 | 收益依赖窗口行数与接受率。 | [src/kernels/cuda/window_gemm_nvfp4.cu](../../../inferences/basalt/src/kernels/cuda/window_gemm_nvfp4.cu) |
| 持久化与多级预取 | dense_tc 的 persistent block 跨 tile 工作，cp.async 环提前取 chunk；固定求和次序。 | 此文件的核心可见 INT8 MMA 指令，不能因为项目叫 NVFP4 就给它错误归类。 | [src/models/qwen4exp/kernels/dense_tc.cu](../../../inferences/basalt/src/models/qwen4exp/kernels/dense_tc.cu) |
| 专家缓存与内存互补 | 计算自动驻留大小、RAM complement、peer tiers；不必在 RAM 重复保存已独占在 peer 的专家。 | 驻留大小影响 CPU/GPU 的舍入路径，A/B 比较须锁 placement。 | [src/models/qwen4exp/load/resident_plan.cpp](../../../inferences/basalt/src/models/qwen4exp/load/resident_plan.cpp) |
| Prefill 临时借用 cache | prefill 可借末端专家 slots；借出上限与 pinned share 相关，自动 chunk 受 context/预算限制。 | 更大 loan 会增加再次流入的专家，不能只追最大 chunk。 | [src/models/qwen4exp/load/resident_plan.cpp](../../../inferences/basalt/src/models/qwen4exp/load/resident_plan.cpp) |
| 权重 staging 与 peer 来源 | copy stream 预先搬 blob；resident、pinned、peer-only、需要 host staging 的来源分别处理。 | P2P 是否有效取决于实际主板和拓扑。 | [src/models/qwen4exp/prefill/expert_stream.cpp](../../../inferences/basalt/src/models/qwen4exp/prefill/expert_stream.cpp) |
| CPU 原生 NVFP4 路径 | AVX2、VNNI、AVX512 对 host 专家权重做低位宽计算。 | CPU 是 MoE miss lane；dense 模型不因此获得通用 CPU-only 主路径。 | [src/kernels/cpu/nvfp4_avx2.cpp](../../../inferences/basalt/src/kernels/cpu/nvfp4_avx2.cpp)、[src/kernels/cpu/nvfp4_avx512.cpp](../../../inferences/basalt/src/kernels/cpu/nvfp4_avx512.cpp) |
| 多卡 peer 专家与持久窗口 | 额外卡可作为专家 tier，window 中 launch peer jobs，结束时同步和诊断。 | 这是 expert placement/执行模式，与简单把层均分给多卡不同。 | [src/models/qwen4exp/decode/verify/expert_execution.cpp](../../../inferences/basalt/src/models/qwen4exp/decode/verify/expert_execution.cpp) |
| 投机、图与采样整合 | 按模型提供 MTP/DFlash/DSpark 等，验证/提交/图捕获分别实现。 | 不是所有模型使用同一种 drafter；drafter 显存和速度都要计入。 | [src/models/qwen4exp/decode/verify/capture.cpp](../../../inferences/basalt/src/models/qwen4exp/decode/verify/capture.cpp)、[src/models/qwen4exp/decode/mtp/draft.cpp](../../../inferences/basalt/src/models/qwen4exp/decode/mtp/draft.cpp) |
| 严格质量和测量契约 | 要求 kernel 参考对照、batch-width 不变性、逐步成本及 BF16 KLD/PPL；ABBA 锁专家 tier。 | 这些是贡献规范；本次未执行其中的硬件 gate。 | [CONTRIBUTING.md](../../../inferences/basalt/CONTRIBUTING.md) |

## 对 ComfyUI 的借鉴

参考持久 kernel、格式/布局协同、图捕获与缓存借用的约束。先用成熟 GEMM 库确定热点与硬件利用率，再决定是否值得写 sm_120 专用融合核。

## 下次跟踪

重点追踪新 pack 的数值代价、模块占用对驻留专家数的影响、首轮与热轮差异；一项优化必须同时报告 prefill 和 decode。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
