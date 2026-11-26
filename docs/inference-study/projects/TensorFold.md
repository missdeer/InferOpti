# TensorFold：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`84d9b6ba1367eddd4f97655185a597b5f909be6c`
- 提交时间：`2026-11-26T03:17:58+00:00`
- 提交主题：changelog: each line once under Unreleased
- 源仓库：https://github.com/ashhart/TensorFold.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

体量最大的 Zig 原生执行器，模型 family 很多。除 LLM、语音与 vision 外，本地还有 Qwen-Image Metal、MOSS-Image CUDA，以及窄范围 ComfyUI 安装/worker 集成；必须按 family/backend 记录能力。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 统一/独立内存准入 | cuda_family_memory 读取 integrated、device mem、Linux meminfo/zoneinfo、保留量，按模型权重/streams 预算。 | GB10 统一内存不能双计 page cache；信息无法读取会拒绝而非猜容量。 | [zig/src/native/cuda_family_memory.zig](../../../inferences/TensorFold/zig/src/native/cuda_family_memory.zig) |
| Prefix cache 显式容量 | cache_budget.fitUnder 扣除 ready/margin，限制请求 cache 到 room/ceiling；显式 over-cap 是另外选择。 | KV/前缀复用与 Living Weights 是不同概念；memory_controller 的名字不代表显存 allocator。 | [zig/src/native/cache_budget.zig](../../../inferences/TensorFold/zig/src/native/cache_budget.zig)、[zig/src/native/memory_controller.zig](../../../inferences/TensorFold/zig/src/native/memory_controller.zig) |
| CUDA arena 子分配 | 小分配来自少数大 chunk，按 size 回收复用；大分配独立 chunk，避免每 tensor 一次 cuMemAlloc。 | 默认小请求上限 4MiB、chunk 64MiB、512B 对齐；是这个 allocator 的参数，不是通用最优。 | [zig/src/cuda/arena.zig](../../../inferences/TensorFold/zig/src/cuda/arena.zig) |
| 图缓存及内存压力回退 | Graph Cache 按 Key 捕获，最多默认 48；低内存时淘汰至 min_keep 或停止捕获，capture 失败可 eager。 | Body 不能同步/分配/host 读设备；指针 fingerprint 与多 rank 一致性要成立。 | [zig/src/cuda/graph_cache.zig](../../../inferences/TensorFold/zig/src/cuda/graph_cache.zig) |
| 异步拷贝源 custody 预算 | copy_source_budget 对 active/quarantine/callback disposal 的源总字节与节点数计费。 | 等待失败而仍在途的 buffer 也要 charge；不能释放后继续 DMA。 | [zig/src/cuda/copy_source_budget.zig](../../../inferences/TensorFold/zig/src/cuda/copy_source_budget.zig) |
| Metal 驻留集合 | stream_resident 把 checkpoint、scratch、各 cache buffers 加入 residency set，新增 cache 通过 follow 同步。 | 驻留提示/管理和逻辑 tensor 大小是不同指标，失败时需清楚记录 skipped。 | [zig/src/metal/stream_resident.zig](../../../inferences/TensorFold/zig/src/metal/stream_resident.zig) |
| Qwen-Image 静态 scratch | Scratch.of 按 text/image 规划统一 buffer 的 18 个对齐视图，检查乘法/总和溢出。 | 是固定 family 的内存规划，不是任意 ComfyUI 节点 DAG 自动编译器。 | [zig/src/families/qwen_image/model.zig](../../../inferences/TensorFold/zig/src/families/qwen_image/model.zig) |
| Qwen-Image prompt 一次计算 | prompt 保存每个 block 的 prompt K/V；forward 各步只处理 image rows，并取已存 prefix。 | 只对该模型实现的独立 prompt 路径成立；不能推广到双向 joint text-image attention。 | [zig/src/families/qwen_image/model.zig](../../../inferences/TensorFold/zig/src/families/qwen_image/model.zig) |
| Qwen-Image 融合与提交 | mod norm、QK norm/RoPE、SwiGLU、gate add 为专用 kernel；每次 forward 的工作放一个 serial compute encoder/command buffer。 | forward 最后 wait 并在 host/设备之间转换 latent/velocity，仍有每步同步，不能称完全 GPU 常驻 sampler。 | [zig/src/families/qwen_image/model.zig](../../../inferences/TensorFold/zig/src/families/qwen_image/model.zig)、[zig/src/families/qwen_image/kernels.zig](../../../inferences/TensorFold/zig/src/families/qwen_image/kernels.zig) |
| MOSS prompt refiner 与 CUDA scratch | prompt refinement 可预先缓存；joint blocks 中 image 和 prompt rows 仍一起算，统一 scratch 限定峰值。 | 不能把 Qwen-Image 缓存方法直接移植为 MOSS 全部 text rows 跨步不算。 | [zig/src/families/moss_image/cuda_model.zig](../../../inferences/TensorFold/zig/src/families/moss_image/cuda_model.zig) |
| 窄范围 ComfyUI worker | 安装器锁定媒体源码/patch/toolchain，核对 hash，成功后原子切换 runtime.json；QKV 准备只重写 header/复制 payload。 | 本地 README 限 Linux aarch64、GB10 sm_121、BF16；明确拒绝 sm_120，非消费级 RTX 通用即插即用后端。 | [tools/comfyui_runtime/README.md](../../../inferences/TensorFold/tools/comfyui_runtime/README.md)、[tools/comfyui_runtime/tensorfold_runtime.py](../../../inferences/TensorFold/tools/comfyui_runtime/tensorfold_runtime.py)、[tools/comfyui_runtime/prepare_moss.py](../../../inferences/TensorFold/tools/comfyui_runtime/prepare_moss.py) |
| 共享权重的并发/CFG 策略 | run-guide 给出 parallel 准入和部分 family 的 CFG 两状态独立执行，CUDA events 汇合。 | 文档明确 CFG batched 不等于 vectorized batching；按 family 核查，不能推定所有图像路径都有。 | [docs/run-guide.md](../../../inferences/TensorFold/docs/run-guide.md) |

## 对 ComfyUI 的借鉴

可直接参考静态 scratch、数值物化/累加语义对照、最窄 worker 接口和失败不破坏旧 runtime 的交付方式。先验证消费级硬件和目标模型，再谈集成。

## 下次跟踪

本轮重点是通用内存/图与两个图像 family；21K 清单中本项目占 8772 个源/测试/工具文件，未逐行核读所有 family、training、app。下一轮可按具体模型追踪 native/kernel/test 三者。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
