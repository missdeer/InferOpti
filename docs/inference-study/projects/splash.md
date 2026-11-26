# splash：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`d40c250ff248938edebdb91fb6124514e32daeaa`
- 提交时间：`2026-11-24T17:28:00-08:00`
- 提交主题：Fix long conversations on Inkling-2-Mini and Flint: a lane takes back the window tails its states pinned (#842)
- 源仓库：https://github.com/incoai/splash.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

Apple Silicon 原生 C++/Objective-C++/Metal 引擎。统一内存上的预算、请求调度、KV 分页/SSD tier、按设备标定的 kernel 选择是重点；它不是独显 H2D 流水线的同义实现。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 统一内存联合规划 | MemoryPlan 计算 fixed、lane state、KV runway、draft scratch；StreamedMemoryPlan 分配专家/PLE store 与上下文容量。 | 整体 host/Metal 争同一个物理池，不能把可用 RAM 和 GPU 内存相加。 | [runtime/engine/MemoryPlan.hpp](../../../inferences/splash/runtime/engine/MemoryPlan.hpp)、[runtime/engine/StreamedMemoryPlan.cpp](../../../inferences/splash/runtime/engine/StreamedMemoryPlan.cpp) |
| 内存压力治理 | MemoryGovernor 读 macOS host pages、compressor、phys_footprint 与 pressure，决定 reserve/准入。 | 依赖 Apple 平台计数，压缩内存可回收估计不是硬保证。 | [runtime/engine/MemoryGovernor.cpp](../../../inferences/splash/runtime/engine/MemoryGovernor.cpp) |
| 空闲权重归还 | WeightResidency 仅在 engine idle 且超时或显式请求时 release；下次请求渐进 restore，拒绝时等待并记录。 | 恢复时间属于下一次冷/温启动；不能只报释放后的 idle 内存。 | [runtime/engine/WeightResidency.cpp](../../../inferences/splash/runtime/engine/WeightResidency.cpp) |
| 分层分页 KV | KV pool/page tier 与 active context 计费，保留 window/full、较瘦历史或 spill 能力对应的形态。 | 能否 spill 由模型 layout、tier 可写能力和最小 floor 决定。 | [runtime/engine/StreamedMemoryPlan.cpp](../../../inferences/splash/runtime/engine/StreamedMemoryPlan.cpp)、[runtime/engine/KvPageTier.cpp](../../../inferences/splash/runtime/engine/KvPageTier.cpp)、[runtime/engine/KvPool.cpp](../../../inferences/splash/runtime/engine/KvPool.cpp) |
| Prefill/decode 调度 | Scheduler 按 command rows、lane phase、priority 与 decode gap target 分割 prefill。 | 过小 chunk 损失吞吐；不能只追最短一次 decode gap。 | [runtime/engine/Scheduler.cpp](../../../inferences/splash/runtime/engine/Scheduler.cpp) |
| 按实际占用选择 tile/split | KernelSelection 用 cores、waves、格式、行数、register/staged tile 的成本模型选择矩阵乘形态。 | 设备校准与边界表是实现的一部分，不是一个 kernel 通吃。 | [runtime/ops/KernelSelection.cpp](../../../inferences/splash/runtime/ops/KernelSelection.cpp) |
| 压缩布局直接矩阵乘 | Metal prefill int8 与 decode kernels 配套 GGUF 格式，融合准备与小矩阵乘。 | 是否快依赖格式与 rows；BF16/F16 全模型并非 README 的支持目标。 | [runtime/metal/kernels/prefill/gguf_prefill_int8.metal](../../../inferences/splash/runtime/metal/kernels/prefill/gguf_prefill_int8.metal)、[runtime/ops/LinearGguf.cpp](../../../inferences/splash/runtime/ops/LinearGguf.cpp) |
| ANE/GPU FFN 分工与校准 | AneFfnCalibration 测不同 share 和 chunk 的 split 成本；收益不足 5% 不选 split。 | 存在专用能力和启动检查；不能理解为任意 Mac/任意模型可无条件用 ANE。 | [runtime/ops/AneFfnCalibration.cpp](../../../inferences/splash/runtime/ops/AneFfnCalibration.cpp)、[runtime/engine/AneFfnStartup.cpp](../../../inferences/splash/runtime/engine/AneFfnStartup.cpp) |
| 前缀/状态 cache 与写回 | Cache、StateCache、KvCache、WriteBehind 分别管理可复用状态和后台写入。 | 容量必须计入治理；文件格式、模型和状态一致性决定复用安全。 | [runtime/engine/StateCache.cpp](../../../inferences/splash/runtime/engine/StateCache.cpp)、[runtime/engine/KvCache.cpp](../../../inferences/splash/runtime/engine/KvCache.cpp)、[runtime/engine/WriteBehind.cpp](../../../inferences/splash/runtime/engine/WriteBehind.cpp) |
| DFlash2 与多 lane | runtime 按模型组织 draft、selector、attention，调度合并并发请求。 | speculative 的 target 验证不等于其量化模型与 BF16 原模型相同。 | [runtime/ops/DraftSelector.cpp](../../../inferences/splash/runtime/ops/DraftSelector.cpp)、[runtime/ops/DraftAttention.cpp](../../../inferences/splash/runtime/ops/DraftAttention.cpp)、[runtime/engine/Scheduler.cpp](../../../inferences/splash/runtime/engine/Scheduler.cpp) |

## 对 ComfyUI 的借鉴

Mac 后端可借鉴系统压力驱动准入、空闲释放/恢复、按 tile 的实测分派及异构加速器分工的净收益门槛。不要把统一内存省搬运误写为没有带宽/驻留成本。

## 下次跟踪

跟踪 Apple GPU 代际、ANE 校准覆盖、运行时压缩/SSD tier 的读写成本和多请求公平性；DFlash 与 diffusion 的语义分开。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
