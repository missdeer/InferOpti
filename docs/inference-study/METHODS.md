# 跨项目优化方法索引

研究基于 [版本快照](SNAPSHOTS.md)。各项指向的项目笔记给出代码位置、条件及默认值。本表列出观察到的实现类别，不代表各项目对全部模型都支持，也不代表本次测得收益。

## 一、把模型放得下，并避免无用搬运

| 方法 | 观察到的项目与路径 | 优化了什么 | 对 ComfyUI 的迁移边界 |
| --- | --- | --- | --- |
| 热专家 VRAM / RAM / SSD 分层 | [Strata](projects/Strata.md)、[Maya](projects/Maya.md)、[XeMaya](projects/XeMaya.md)、[QuixiInfer](projects/QuixiInfer.md)、[Kinetic](projects/kinetic.md) | 用容量较大的慢层容纳完整模型，只把有价值部分放快层 | MoE 才有稀疏专家热点；dense DiT 每步几乎都访问每层，应用顺序预取而非照搬 LFU |
| 部分层常驻、剩余层 streaming | [XeStrata](projects/XeStrata.md) 的图像/TE 路径；[Strata](projects/Strata.md) 的图像实验 | 减少重复权重传输，同时保持显存峰值有界 | 对小显存 DiT 直接相关，但需为激活、scratch 和预取槽先留预算 |
| 物理容量、外部占用与分配粒度核算 | [XeStrata](projects/XeStrata.md)、[TensorFold](projects/TensorFold.md)、[Splash](projects/splash.md) | 避免“tensor 大小之和能装下，实际分配却 OOM” | 独显、UMA、WDDM、库 workspace 的计数不同；不能只读启动时 free memory |
| 临时区借用空闲 cache / slot 尾部 | [Strata](projects/Strata.md)、[Basalt](projects/basalt.md)、[QuixiInfer](projects/QuixiInfer.md) | Prefill 或另一个阶段复用暂不需要的容量 | 扩散可在 TE、DiT、VAE 阶段间借还；须有唯一所有者和完成事件 |
| 稳定虚拟地址、弹性物理映射 | [Strata](projects/Strata.md) 的 VMM | 改变实际占用时维持图内指针有效 | 依后端能力；与 PyTorch allocator 共存需专门设计，不能双重管理同一 storage |
| host 锁页预算与退让 | [Strata](projects/Strata.md)、[Maya](projects/Maya.md)、[QuixiInfer](projects/QuixiInfer.md) | DMA 快且不饿死必需 staging | 锁页不是越多越好；Windows 配额与整机 RAM 压力都要观察 |
| 空闲释放、逐步恢复和内存压力准入 | [Splash](projects/splash.md)、[TensorFold](projects/TensorFold.md) | 给桌面其他程序让内存，防止过度并发 | 下一次恢复会产生额外延迟；报告 idle RAM 的同时要报告恢复耗时 |
| mmap 与实际页驻留控制 | [Cheetah](projects/Cheetah.md) native、其他文件 backing | 避免全量堆拷贝，按需驻留 | mmap 不等于已在 RAM，DONTNEED 不等于清 OS cache，更不等于 GPU 已卸载 |

## 二、让搬运尽量和计算重叠

| 方法 | 项目 | 必需的正确性条件 |
| --- | --- | --- |
| 单独 copy/compute 队列、双缓冲/环形槽 | XeStrata、Strata、Maya、Basalt | copy 等上次消费，compute 等上传完成；host staging 也不能提前复用 |
| 文件直读 pinned staging | XeMaya、QuixiInfer | 读取完成才提交 DMA；DMA 完成才发布 residency |
| 多线程填大层、批量读取/填槽 | XeStrata、XeMaya、Maya | 避免增加过多线程抢主存带宽；读取顺序与下游消费一致 |
| O_DIRECT / io_uring / unbuffered read | QuixiInfer；Maya/XeMaya 部分路径 | 大小和地址对齐，处理部分读/失败；必须观察后续页缓存命中是否受损 |
| 只传路由实际需要的数据 | Strata 可选 routed_stream | 只有路由已知且尚未发出的条目才能跳过；事件消费者不能等一个不会发布的事件 |
| 合并连续上传、避免小拷贝 | Maya、Strata 的 KV DMA | 相邻内存与源生命周期合法，不能破坏重叠与优先级 |
| 直接 mapped RAM 计算 | QuixiInfer / 分层引擎的部分路径 | 省一次复制不代表省 PCIe 读；重复访问可能比先 DMA 再本地算更慢 |
| 就近 NUMA、peer device | Maya、Basalt、XeMaya | 检查拓扑、P2P、共享 PCIe 根端口、host RAM 带宽，不能只看显存总和 |

XeMaya 的项目记录给出一个值得保留的负面经验：绕过文件页缓存可改善 prompt 读取，却使后续 CPU 专家 decode 变慢。**优化一个阶段时，要看完整任务。**

## 三、压缩格式、计算精度与 kernel 一起设计

| 方法 | 源码参考 | 收益来源与代价 |
| --- | --- | --- |
| 量化存储、执行前展开 FP16 | Strata 图像实验 `dit_gemm.cu` | 存储/传输小，但需要展开 scratch 与转换时间，不自动获得 INT4/FP4 算力 |
| 压缩权重内算、MMQ | Sync-Strata、Speedy、Kinetic、YASAKANI、Basalt | 少读权重且不全量展开；多出解码/LUT/scale 成本，随 batch/形状变化 |
| W8A8、旋转和逐行 scale | XeStrata DiT；多个 LLM 路径 | 激活与权重以目标整数格式进 GEMM；量化、逆 scale 及质量需一起验收 |
| NVFP4 planes / 专用 pack | Basalt | 与特定硬件和 kernel 契约绑定；pack 名不代表所有 kernel 都在用 FP4 MMA |
| trellis / codebook / 混合率 | Sync-Strata、YASAKANI | 低位宽与误差之间的细分选择；格式和解码器紧耦合，不能当通用 INT2 |
| 一次量化供多个投影复用 | Strata、XeStrata | 相同输入的 Q/K/V 或 gate/up 避免重复准备；输入修改或中间算子介入就不能复用 |
| 融合 norm、RoPE、modulation、gate、bias/scale | XeStrata、TensorFold、Strata 系列 | 减少 kernel launch 与临时读写；融合过度可能增加寄存器/共享内存压力 |
| 小行数 GEMV 与大行数 GEMM 分路 | Basalt、Kinetic、Splash、Strata 系列 | decode 与 prefill 的矩阵形状不同；DiT 的空间 token 矩阵更接近大 GEMM |
| persistent block、异步共享内存预取 | Basalt、YASAKANI | 少启动、复用权重/tile、隐藏设备内存延迟；硬件与 occupancy 限制强 |
| Flash/tiled attention 与在线 softmax | Maya、XeStrata、Kinetic 等 | 避免全量 attention score 中间矩阵；mask、累加精度和选取集合仍需保持 |
| MLP 逐 token band、VAE 逐空间 tile | XeStrata | 峰值显存变小；执行次数增加，VAE tile 可能改变全局运算效果并出现接缝 |

**权重量化、激活量化、KV 量化是三类不同改动。** 同样写 INT8，质量影响、显存收益和 kernel 命中率都可能不同。文件 format、scale/offset、layout、转换和 GEMM 必须沿调用链一起检查。

## 四、少做重复工作与减少提交开销

| 方法 | 项目 | 能否直接移植到 diffusion |
| --- | --- | --- |
| prompt/条件编码缓存 | 多数 LLM 引擎；图像路径也有 | 可迁移，但 key 要包含输入、模型、适配器、预处理、精度等真实依赖 |
| 各层 prefix K/V 缓存 | XeStrata Qwen 图像、TensorFold Qwen-Image | 仅当 prefix 的状态不依赖随 timestep 变化的图像状态且 mask 支持时成立 |
| checkpoint/park/resume | Strata、Kinetic、QuixiInfer、TensorFold | 可借鉴完整状态定义；不能只保存 latent 而遗漏 scheduler/RNG/控制状态 |
| CUDA Graph/形状缓存 | Strata 系列、Basalt、Kinetic、TensorFold、YASAKANI | 可迁移；要稳定地址、形状和分配，动态 LoRA/条件变化使图失效 |
| CPU 原生运行库减少 Python 派发 | 所有 native engine | 仅优化主机开销，不保证主耗时 GEMM 更快；要先看墙钟/设备时间差 |
| 连续批处理/共享 weight pass | QuixiInfer、Kinetic、Splash、TensorFold、YASAKANI | 可借鉴同模型/尺寸/调度状态分桶，显存增长和 p95 必须计入 |
| GPU 上 greedy / 限制 logits 下载 | Kinetic 等 | 主要是文本模型优化；不能直接声称会加速 DiT |

## 五、减少 target 调用次数的 LLM 专属方法

MTP、DFlash/DSpark、n-gram/suffix lookup、copy-from-input 和编辑规则在多项目中出现。共同机制是：产生 proposal → target 批量验证 → 接受前缀 → 回滚其余状态。接受长度、draft 时间、验证行数、额外缓存和 miss 专家数共同决定净收益。Strata 的 controller 明确比较 tokens/ms，而不是只追接受率。

“验证后 exact”通常指**相对于同一量化模型、相同后端和同样采样语义的普通执行**；不代表与 BF16 原模型相同。批次宽度改变浮点归约、混合 CPU/GPU placement 改变舍入，都可能破坏该条件。

扩散 timestep 是连续状态更新，不能套用 token 接受前缀机制。减少步数、换蒸馏模型或跨步近似复用，必须作为新的算法/质量实验，不能标成移植 LLM 投机即可无损提速。

## 六、经验规则

1. **先识别容量、带宽、算力、提交和重复工作中哪一项限制实际任务。** 消费级 LLM 单 token decode 常表现为读权重/CPU miss 成本；图像 DiT 的大量空间 token 可能使矩阵乘主导。不能跨任务套瓶颈。
2. **能运行、低显存、快、吞吐高、首张快是不同目标。** SSD streaming 可以让模型运行，却可能减慢每步；合批提高聚合吞吐，也可能增加单请求延迟。
3. **策略应有触发条件和退路。** 缺 pinned、DSO ABI 不匹配、图捕获失败、workspace 不足、能力缺失时，回到已知路径或明确拒绝；不能无声空跑。
4. **负面结果同样入库。** 记录“微核快但端到端无变化”“prefill 快但 decode 慢”“命中率高但 promotion 太贵”“图更少但占用更多”，防止后来重复尝试。
5. **开关表不能取代调用链。** opt-in 代码、测试入口、未发布程序、正式默认路径分别标明。相同源码在不同分支也可能有不同默认。

## 七、tqLLM 增量经验

[tqLLM笔记](projects/tqLLM.md)新增38项方法，完整版本见[增量快照](updates/2026-11-26-tqLLM/README.md)。尤其补充以下细节：

| 问题 | tqLLM中的实现 | 可积累的经验 |
| --- | --- | --- |
| native量化算子是否一定优于BLAS | 对比batched matvec、native GEMM、dequant+cuBLAS的固定/逐行成本交点 | 以格式、M/N/K、设备和scratch分派；少展开不保证对大矩阵更快 |
| 压缩率与实际格式 | TQ4/TQT都可为4.125bpw，codec不同；TQ3_4S文件块为32权重16字节 | 总字节含scale和布局，名字不能代替格式；低位权重不等于低位MMA |
| 自动fit是否保持比较条件 | fit_lossy默认true，可公告KV/context/prefill改动；exact显式近似选项可获胜 | 记录最终生效配置；容量策略与画质/数值策略单列 |
| 预取是否占满需求链路 | 专家预测park有限、demand优先、控制排队深度、多请求减少猜测范围 | 评价隐藏了多少等待和浪费了多少字节，不能只看预测命中 |
| host与GPU是否应该总去重 | host tier压力下优先回收GPU已持有块，宽裕时保留副本避免D2H | 容量和搬运的权衡随RAM压力变，不采用无条件exclusive原则 |
| 动态策略是否覆盖所有入口 | Inkling驻留重规划同时统计plain与verify，稳定后退避 | 首次请求也要验收；投机/控制分支不能绕过资源策略 |
| 在线调优能否改变答案 | 数值不同的kernel固定设备/形状规则，避免重启随测速噪声选族 | 先验证候选数值等价性，再考虑自动择优 |
| 图为何在低内存反复重建 | GraphLru有上限、hot floor、销毁前quiesce；owner ID避免地址重用 | 捕获图需自己的预算/身份/生命周期，不能只做无界形状字典 |
| 缓存与prefill宽度的关系 | 按chunk和消息anchor保存状态，exact下不任意复用live decode末尾 | 分块/归约顺序也可能属于复现契约 |

这些机制补充之前的分类，不改变LLM与diffusion的适用边界。QSA、KV compact、token draft和vocab筛选不能直接当作通用DiT优化。
