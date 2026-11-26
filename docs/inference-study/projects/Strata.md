# Strata：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`c234774038d1c9249cb33637d27b895d47b39786`
- 提交时间：`2026-11-25T23:59:41+01:00`
- 提交主题：fix(packaging): bundle libgomp.so.1 with the engine, like libatomic.so.1
- 源仓库：https://github.com/Niko1221/Strata.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

这一组项目的重要共同基础。主体是消费级机器上的 MoE LLM 引擎，CPU 专家池、GPU 常驻专家、量化算子、投机验证与服务端状态缓存共同组成执行路径。当前目录已重组为 engine、terminal、app；旧文档中的 src/ 应在 engine/src/ 下定位。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 专家分层与热点常驻 | ExpertCache 保存 (layer, expert) 到 slot 的映射；按层配置配额，支持初始频率排名、运行热度和学习后 profile 保存。避免最先访问的层吃光全局缓存。 | 缓存本体不等于替换策略；admit 本身不淘汰，替换由上层协调。 | [engine/include/strata/core/expert_cache.hpp](../../../inferences/Strata/engine/include/strata/core/expert_cache.hpp)、[engine/src/core/expert_cache.cpp](../../../inferences/Strata/engine/src/core/expert_cache.cpp) |
| 显存弹性与稳定地址 | 分段 CUDA VMM 映射可缩小/扩展缓存物理页，保留虚拟地址供已捕获的图使用。 | --vram-elastic 可选；缩小前调用方必须清掉尾部专家和借用区；HIP 不支持此路径。 | [engine/include/strata/core/expert_cache.hpp](../../../inferences/Strata/engine/include/strata/core/expert_cache.hpp)、[engine/src/core/vmm.cpp](../../../inferences/Strata/engine/src/core/vmm.cpp) |
| CPU/GPU 专家分工 | 命中专家在 GPU 算，未命中在 CPU 池算；CPU 路径按量化格式和 AVX 能力分派，主线程和池的工作通过同步协议汇合。 | 残差链仍串行；并行不能把整个 CPU 代价都隐藏。 | [engine/src/core/verify.cpp](../../../inferences/Strata/engine/src/core/verify.cpp)、[engine/src/kernels/cpu/pool.cpp](../../../inferences/Strata/engine/src/kernels/cpu/pool.cpp) |
| Pinned 上传环与生命周期 | PinnedRing 用四个槽及各自完成事件上传临时源；槽复用前等上次 DMA 完成；无 pinned 空间时同步回退。 | 解决异步源被提前修改/释放的问题；不是只加 non_blocking 标志。 | [engine/include/strata/core/pinned_ring.hpp](../../../inferences/Strata/engine/include/strata/core/pinned_ring.hpp) |
| 锁页内存预算 | 在专家 arena 注册前预留 soft/hard pinned 空间；小型必需 staging 分配失败时释放硬预留重试。 | Windows CUDA 默认预留 640+160 MiB；Linux 默认关闭，不能把配置抄到所有平台。 | [engine/include/strata/core/pin_budget.hpp](../../../inferences/Strata/engine/include/strata/core/pin_budget.hpp) |
| Prefill 分块、分组与传输流水 | 大块 prompt 按专家聚合计算；staging 环并行准备权重并上传；可借用专家缓存空间做临时工作区。 | Prefill 和 decode 采用不同路径；块越大，激活与工作区需求越大。 | [engine/src/prefill/prefill.cpp](../../../inferences/Strata/engine/src/prefill/prefill.cpp) |
| 避免无用专家传输 | routed_stream 在已知路由后跳过尚未发出的未使用专家，限制拷贝排队深度。 | STRATA_PREFILL_ROUTED_STREAM 默认关闭；不适用于 peer issuer 或不兼容的事件策略。 | [engine/include/strata/prefill/routed_stream.hpp](../../../inferences/Strata/engine/include/strata/prefill/routed_stream.hpp) |
| KV 压缩、分层与批量 DMA | 存在 Q8/Q4 KV 路径；流式 KV 的新数据可先写设备 staging 再整块 DMA 回 host。 | KV_HOST_DMA 的 CUDA 默认开、HIP 默认关；精度变动与传输优化要分别验收。 | [engine/src/core/layer.cpp](../../../inferences/Strata/engine/src/core/layer.cpp)、[engine/include/strata/core/kv_host_dma.hpp](../../../inferences/Strata/engine/include/strata/core/kv_host_dma.hpp) |
| 融合和量化布局契约 | 复用同一输入的激活量化；融合 norm、门控等；GEMV 按张量的真实 source_type/scale/offset 选择解码。 | 同名权重在不同层可能使用不同量化，不能根据节点名或位数猜格式。 | [engine/src/core/layer.cpp](../../../inferences/Strata/engine/src/core/layer.cpp) |
| 图捕获与重复执行 | GraphRegistry 按层类型和 token 数缓存 CUDA Graph；拒绝空捕获并检查异步错误。 | 图需要稳定地址与形状；不能与任意卸载/缓冲重分配独立设计。 | [engine/src/core/graph.cpp](../../../inferences/Strata/engine/src/core/graph.cpp) |
| 自适应投机长度 | CostModel 计入 dense、CPU miss、同步和 draft 成本；Controller 用接受率 EMA 选择 lookup/MTP 及长度，无净收益则关闭。 | 该 controller 是可见实现；不能假定所有模型入口都以相同策略调用它。 | [engine/src/spec/controller.cpp](../../../inferences/Strata/engine/src/spec/controller.cpp) |
| 会话状态与前缀复用 | 快照包含注意力状态、递归状态、几何和图像 key 校验，支持继续已有前缀。 | 不仅是 token KV；GDN/SSM 的回滚也要恢复。 | [engine/src/core/conversation_state.cpp](../../../inferences/Strata/engine/src/core/conversation_state.cpp) |
| 原生图像实验路径 | Qwen-Image 文本编码→DiT→Euler→Wan VAE；小显存时先编码后释放 TE；DiT 量化权重先还原 FP16 后 cuBLAS 计算，复用 scratch。 | qi21_image_main 明说测试程序未交付、server 不调用，sd-cli 仍默认；并非已商用替换后端。 | [engine/src/image/qi21_image_main.cpp](../../../inferences/Strata/engine/src/image/qi21_image_main.cpp)、[engine/src/image/dit/dit_gemm.cu](../../../inferences/Strata/engine/src/image/dit/dit_gemm.cu) |

## 对 ComfyUI 的借鉴

优先借鉴带事件的预取、显存/锁页双预算、阶段工作区借用及图地址稳定性。MoE 热专家策略仅适用于实际有专家路由的目标模型；dense DiT 更适合顺序层驻留和预取。原生图像样例还展示了“量化存储不一定等于低精度矩阵乘”。

## 下次跟踪

继续追踪新的 engine 布局、routed_stream 默认值、CUDA/HIP parity、图像入口是否正式接到 server；分开记录启动、首图、后续图与每步搬运量。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
