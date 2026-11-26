# ComfyUI 后端：源码借鉴与待验证清单

这是由本轮源码研究得到的迁移建议，不是已实现功能或性能承诺。各项目固定版本见 [SNAPSHOTS.md](SNAPSHOTS.md)，具体机制和限制见 [项目笔记](README.md)。

## 1. 先对照本地 ComfyUI 基线

实际代码在 `ComfyUI/ComfyUI/`，HEAD 为 `a277b5bf213e8a44fa34817b7770b54f86ef4cea`，本次读取时 tracked clean、untracked 0。这是本地 checkout 的事实，不代表所有 ComfyUI 发布版都有以下行为。

| 本地已存在的机制 | 源码 | 后续应比较什么 |
| --- | --- | --- |
| 异步 weight offload，多 stream；NVIDIA/AMD 未显式配置时 NUM_STREAMS=2 | [model_management.py](../../ComfyUI/ComfyUI/comfy/model_management.py) | 真实模型是否使用、copy 是否被隐藏、额外缓冲是否挤出权重 |
| 预取队列、block pin、权重 residency signature | [model_prefetch.py](../../ComfyUI/ComfyUI/comfy/model_prefetch.py) | 队列深度、首 miss/热 hit、层大小差异和 OOM 时的行为 |
| 分块 CUDA Graph、warmup、signature 核验、失效清理、allocator graph scope | [model_prefetch.py](../../ComfyUI/ComfyUI/comfy/model_prefetch.py) | 多少 block 能捕获、graph break、shape/LoRA 变化、capture 内存代价 |
| cast buffer、combined linear、量化布局准备与低显存 patch | [ops.py](../../ComfyUI/ComfyUI/comfy/ops.py) | 是否重复转换、联合投影是否命中、动态 patch 是否阻断融合 |
| tensor geometry、文件 slice 与直接填目标 buffer | [memory_management.py](../../ComfyUI/ComfyUI/comfy/memory_management.py) | 是否重复 host 拷贝、文件缓存/pinned 的实际占用 |
| ModelPatcher 与低显存模型生命周期 | [model_patcher.py](../../ComfyUI/ComfyUI/comfy/model_patcher.py) | 外部后端怎样保留 LoRA、patch、卸载、取消和重新加载语义 |

所以“给 ComfyUI 新增异步加载/图捕获”不是合适的未经测量目标。应先量现有路径，在同样输入与质量下验证候选差异。

## 2. 最值得迁移的内容

| 优先级 | 候选 | 首要参考 | 最小实验 | 何时有价值 |
| --- | --- | --- | --- | --- |
| P0 | 建立真实阶段、copy 和内存账本 | Strata/Basalt 的分阶段测量、XeStrata 预算 | 同一 workflow 记录加载、TE、每步 DiT、VAE、输出；区分 CPU prepare、H2D、GPU compute | 先确定 bottleneck，避免重写已经优化的路径 |
| P1 | 分阶段驻留和 scratch 复用 | XeStrata ImagePipeline、TensorFold Scratch | 固定 TE→DiT→VAE，记录每阶段 live/peak 与可借还容量 | 模型换入换出或中间激活造成明显峰值 |
| P1 | 可测量的层预取策略 | XeStrata LayerStream、Maya cost model | 现有预取为对照，比较常驻层份额、1/2 槽、prepare 提前量 | GPU 等权重而空闲，且有足够 workspace |
| P1 | 减少 host 中转与转换 | XeMaya read_blobs、Strata PinnedRing | 测 mmap→relay→GPU 与直接填 pinned，记录 host memcpy 和 DMA | 主线程准备时间或 pageable staging 暴露明显 |
| P1 | 确定性条件缓存 | XeStrata/TensorFold 的条件路径 | 先缓存可证明独立的编码结果；再考虑模型内 prefix | 重复 prompt/参考资产，编码或 prefix 占比足够 |
| P2 | 大 MLP 的 band/scratch 规划 | XeStrata QiDit | 固定图像尺寸比较全量与分带的峰值和总时间 | activation 主导 OOM，能以少量时间换更高分辨率 |
| P2 | 权重格式与实际 GEMM 对齐 | XeStrata DitProducts、Basalt、Sync-Strata | 对一个 Linear 比较 FP16/BF16、量化后展开、压缩 MMQ/W8A8 | 目标硬件/尺寸存在更快可用核，质量可接受 |
| P2 | 消费端融合 scale/bias/norm/gate | XeStrata DitOut、TensorFold image kernels | 保持主要 GEMM 不变，只融合确认的相邻小算子 | profiler 显示临时读写或 launch 开销占比高 |
| P2 | 图捕获与卸载联合规划 | Strata VMM、TensorFold Graph Cache、本地 ComfyUI | 固定尺寸桶，测捕获命中/占用、地址失效与 eager fallback | CPU 提交明显，重复形状多且地址能稳定 |
| P2 | VAE 自适应 tile | XeStrata QiVae | 对照现有 VAE tiling，测 overlap、tile shape、质量 | 高分辨率 VAE 峰值成为限制 |
| P3 | 跨任务共享模型 pass | Kinetic Convoy、QuixiInfer、Splash | 同模型/尺寸/step 状态下的小批任务，限制等待 | 多用户服务且能满足内存和 p95 预算 |
| P3 | 多卡/CPU/SSD 混合执行 | Maya、Basalt、XeMaya、QuixiInfer | 先测链路和串行模型路径，逐个设备加入 | 明确的容量需要或可隐藏通信，收益覆盖复杂度 |

P0–P3 是建议顺序，不表示每项都要实现。上一项未发现瓶颈时，应跳过对应改造。

## 3. 为什么 dense DiT 不能照搬 MoE 热专家缓存

MoE 每 token 只用少数专家，热点缓存可直接减少慢层读取。Dense DiT 每个 denoising step 通常要执行全部 transformer block；用“最近最热”替换层权重不会像稀疏专家一样跳过大量计算。更自然的方案是：

1. 先给激活、GEMM/attention scratch、预取槽和其他活跃模型留空间。
2. 能常驻的层留在 VRAM，剩余层按确定顺序预取；避免每个 step 重新解码/重排同样的权重。
3. 使用完成事件管理两个槽；没有足够计算可重叠时，接受传输是下限，不用更多 stream 掩盖问题。

在理想稳态流水线下，可用粗略下界评估是否值得做：

`每步时间 ≥ max(本步 GPU 计算时间, 本步非驻留权重字节 / 实测上传带宽)`

这只是下界；首尾填空、文件读取、host prepare、同步和依赖会增加时间。假如每步非驻留权重为 6 GiB，实测有效上传只有 12 GiB/s，则仅传输下限约 0.5 秒/步。写更快的 CPU 编排器不能消除这些字节；压缩、驻留或减少访问次数才会改变该下限。这是算例，不是任何项目实测。

## 4. 缓存要按数学依赖分类

| 内容 | 何时可缓存 | 失效条件示例 |
| --- | --- | --- |
| 预处理结果 | 确定性且所有输入固定 | 原图/蒙版、尺寸、颜色空间、预处理器版本/参数变化 |
| TE 编码 | prompt、模型、tokenizer、适配器、精度固定 | 模型或 LoRA/clip-skip/截断/提示词权重变化 |
| reference latent | VAE 和 reference 预处理固定 | 图像、裁剪、VAE、归一化、分辨率改变 |
| block 内 prefix K/V | 必须证明 prefix 状态不依赖 timestep 或动态 image stream | 参考图、mask、LoRA、ControlNet、上下文分支变化 |
| denoiser 中间结果跨步复用 | 一般需要新的误差控制或特定模型数学保证 | timestep、latent、CFG、控制条件变化；默认按有损候选处理 |

XeStrata 的代码明确在 ControlNet 激活时重算相关 prefix、LoRA 改变时丢弃缓存；TensorFold 的 MOSS joint blocks 又不同于其 Qwen-Image prefix。**“文本没变”不是所有 DiT 内部状态不变的充分条件。**

持久化或跨用户缓存还应有命名空间和大小上限；不要让某个大参考图占满所有请求预算。缓存命中收益、维护成本和缓存占用须同时记录。

## 5. 接入边界建议

初期保留 ComfyUI 的工作流、节点和 ModelPatcher 语义，只为一个固定模型段提供可对照的计算实现。要明确谁拥有权重、CUDA context/stream、allocator 和输出 tensor；不能在 Python 与原生库各存一份大模型再宣称低显存。

可选边界包括 PyTorch 自定义算子、模型 forward 适配或独立 worker。根据实际热点选最窄边界，不预先建设通用 IR/多后端框架。若必须经过 worker 文件/CPU IPC，成本计入端到端；跨进程并不能天然零拷贝共享 GPU tensor。

TensorFold 的本地 ComfyUI runtime 是有用的交付参考，但当前只限定 GB10 sm_121 和 MOSS BF16，且媒体源码固定到不同于本次 TensorFold HEAD 的 `1ddac962008367ec2c50e10202c910ecf430b5eb`。本轮没有下载该历史源包。安装器、HEAD 的模型代码、兼容 patch 和实际安装 worker 要分别定位，不能混成同一份实现。

商业实现引用代码前按具体文件检查许可：尤其 Speedy 自有贡献的 Non-Commercial、XeStrata/XeMaya 的 LGPL 及各引入文件声明。这里记录本地许可证事实，不代替具体交付方案的许可核查。

## 6. 基准与验收

每次只改变一项策略，固定模型 hash、源码 commit、节点版本、输入/seed、分辨率、步数、sampler、CFG、LoRA/ControlNet 和驱动。至少包含以下不同工作负载：

- 全模型可驻留、显存刚好不足、明显需要 streaming。
- 首次冷启动、模型已加载但条件未缓存、重复条件热请求、换尺寸/适配器后请求。
- 单张低延迟与多请求吞吐分别统计；不要以热缓存/多人聚合值冒充首次单图速度。

记录墙钟与设备时间、每步分布、H2D/D2H 字节、prepare 时间、cache hit、峰值 driver VRAM、allocator 已用/预留、host RSS/pinned、失败/OOM、功耗和重试。对 streaming 还应记录每步重复搬运字节，证明收益来自少搬/隐藏，而非误跑或漏算。

质量分两级：保持数学和精度的改动先比较关键中间 tensor 误差和最终图；量化、步数、蒸馏、tile 和近似缓存单独列实验，人工检查人脸身份、衣服文字/花纹、颜色、边缘与姿态。视频另测时序稳定和接缝。相同 seed 不保证跨后端逐位相同，也不应把“不逐位相同”直接等同于可接受质量。

异步/缓存改动另测：取消、异常、低 pinned 预算、连续变尺寸、LoRA 切换、ControlNet 开关、重复运行和并发。目标是没有旧权重/旧条件污染、没有覆盖在途 buffer，也没有随请求增长的内存滞留。

没有真机记录前，候选状态保持“待验证”，不填预计倍数。已有技术路线中的理论目标也不能改写为本轮实测。

## 7. tqLLM 带来的补充候选

新增[tqLLM研究](projects/tqLLM.md)，源码固定到`7df840632cd3f108de459e0944b9b6d115532689`。没有据此修改ComfyUI源码。

- **先为一个DiT的Linear建立格式/形状交点。** 比较native压缩GEMM与反量化+现有BLAS，单列反量化固定成本、scratch峰值及激活token数；不要按“新算子”三个字选路径。
- **给最终配置留证据。** tqLLM的exact与fit_lossy揭示两个独立维度：数值计算路径和装载退让。ComfyUI的严格对照应固定尺寸、步数、精度、adapter和控制条件，任何退让另列质量实验。
- **检验所有forward入口的策略一致性。** 首图、投机/控制分支、并发和长输入必须使用正确的驻留更新；稳定后减少更新次数，以免维护开销抵消收益。
- **host cache按压力权衡副本与去重。** 对已经重排/准备好的块，保留host副本可避免GPU淘汰时再D2H；RAM紧张时又该回收副本。分别测首轮/热轮和H2D/D2H。
- **图缓存要有上限和热工作集。** 参考GraphLru的floor与owner身份，再对照本地ComfyUI已有signature与失效机制，观察是否反复捕获、误用旧参数或增加显存峰值。

tqLLM的下一层MoE专家预测不应替代dense DiT的确定性顺序预取。TQT离线校准若用于图像模型，需要图像任务及timestep对应激活，不沿用LLM文本校准集；低位KV、QSA、copy draft也没有直接通用迁移保证。
