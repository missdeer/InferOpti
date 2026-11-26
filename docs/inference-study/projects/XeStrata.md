# XeStrata：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`aadd4f9fea9b5b319855826f6a33c72dce5b529e`
- 提交时间：`2026-11-25T22:51:50+09:00`
- 提交主题：Windows: cudnn-frontend の NVRTC も import library でつなぎ遅延ロードする
- 源仓库：https://github.com/MistVVK/XeStrata.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

Strata 衍生的 SYCL 原生执行器；以设备能力而非单个品牌型号选路径。当前源码有直接相关的图像引擎，涵盖 DiT、文本编码、VAE、ControlNet 与适配器。图像代码是本轮最值得深入借鉴的部分。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 部分层常驻与双缓冲流式权重 | LayerStream.configure 先预留一/两个最大层槽，再把剩余预算给常驻层；worker 在线程中填充，begin 提前请求下一层。 | 全权重可驻留时不流式；不足最小层槽时报错，不能凭空支持任意小显存。 | [src/image/qi_stream.cpp](../../../inferences/XeStrata/src/image/qi_stream.cpp) |
| 拷贝和计算双向事件 | copy 等 slot.used，compute 等 slot.ready；end 后才允许覆盖。 | 预取必须同时保护 host relay 与 device slot，取消/销毁时排空队列。 | [src/image/qi_stream.cpp](../../../inferences/XeStrata/src/image/qi_stream.cpp) |
| 实测选择 host 来源 | 比较 mmap 文件页直接上传与 page-locked home；按实际设备拷贝能力和耗时选源；并行 memcpy 填充大层。 | 选 file 的条件是耗时不超过 locked 的 1.25 倍；不支持 mapped-file 拷贝的后端直接排除。 | [src/image/qi_stream.cpp](../../../inferences/XeStrata/src/image/qi_stream.cpp) |
| 准备结果跨步复用 | 已 prepare 的权重保存在 host home，下轮仅 DMA；home 延迟建立，保留 RAM 余量。 | 会增加 host 常驻内存；预算缩小时可 drop_homes。 | [src/image/qi_stream.cpp](../../../inferences/XeStrata/src/image/qi_stream.cpp) |
| 分阶段真实显存预算 | ImagePipeline 计算 held/free/fixed/scratch，扣除 runtime、库、外部进程和驱动分配粒度变化。 | 总张量字节不等于驱动物理占用；预算要在运行阶段重新核对。 | [src/image/qi_pipeline.cpp](../../../inferences/XeStrata/src/image/qi_pipeline.cpp) |
| 共享 W8A8/原生 GEMM | DitProducts 共用 FP16/native/W8A8 入口；复用同一激活的量化；支持延迟 scale/bias 到后续消费者融合。 | LoRA/DoRA/OFT 等会改变可融合条件，带动态 delta 时需要正确回退。 | [src/image/dit_product.cpp](../../../inferences/XeStrata/src/image/dit_product.cpp)、[src/image/dit_rows.hpp](../../../inferences/XeStrata/src/image/dit_rows.hpp) |
| 能力分派和通用算子复用 | Gemm 在 oneMath 可用时调用对应 BLAS，否则用自有 XMX/FMA 等；Attention 使用 matrix tiles、在线 softmax。 | 有 SYCL 不代表所有 GPU 同速；缺矩阵能力时必须有备用路径。 | [src/prefill/gemm.cpp](../../../inferences/XeStrata/src/prefill/gemm.cpp)、[src/kernels/xe/flash_attn.cpp](../../../inferences/XeStrata/src/kernels/xe/flash_attn.cpp) |
| DiT 的确定性前缀缓存 | Qwen 图像路径保存各 block 的 prefix K/V，后续步只算图像 query；缓存由 prefix mask/上下文结构支持。 | ControlNet 激活时重算；LoRA 变化 drop_prefix；不是所有扩散模型都能缓存任意 text K/V。 | [src/image/qi_dit.cpp](../../../inferences/XeStrata/src/image/qi_dit.cpp) |
| MLP 分带与融合 | MLP 逐 band 处理宽中间张量；融合 LN+modulation、QK norm+RoPE、gate+add，减少临时 tensor 和 kernel 数。 | 带宽、寄存器和 activation 峰值需一起看；分得过小也会变慢。 | [src/image/qi_dit.cpp](../../../inferences/XeStrata/src/image/qi_dit.cpp)、[src/image/dit_product.cpp](../../../inferences/XeStrata/src/image/dit_product.cpp) |
| VAE 自适应 tile | scratch 足够走整图，否则在预算内选择 tile，GPU crop、overlap 加权 blend、finish 归一化。 | 含全局 attention/归一化的 VAE 分块不保证与整图逐位一致，需测接缝与质量。 | [src/image/qi_vae.cpp](../../../inferences/XeStrata/src/image/qi_vae.cpp)、[src/image/tile_sides.cpp](../../../inferences/XeStrata/src/image/tile_sides.cpp) |
| LLM 分层与 CPU fallback | 保留专家缓存、CPU 专家和多设备 plain model；与图像引擎共用产品/attention 基础件。 | MoE 专家缓存与 dense 图像层 streaming 是不同使用模式。 | [src/core/plain_model.cpp](../../../inferences/XeStrata/src/core/plain_model.cpp)、[src/core/expert_source.cpp](../../../inferences/XeStrata/src/core/expert_source.cpp) |
| 设备和驱动约束 | 读取设备内存、矩阵/子组能力，平台代码还处理特定编译器/driver 故障规避。 | 移植时要复验设备/驱动组合，不能只看编译通过。 | [src/core/device.cpp](../../../inferences/XeStrata/src/core/device.cpp)、[AGENTS.md](../../../inferences/XeStrata/AGENTS.md) |

## 对 ComfyUI 的借鉴

优先级最高：把 LayerStream 的策略与 ComfyUI 现有预取逐项比较；再验证条件缓存、MLP 分带及 GEMM epilogue 融合。代码和模型范围较大，先选一个固定 DiT，不要一次承诺所有节点兼容。

## 下次跟踪

记录 budget 计算、source 选择、首轮 prepare 与热步 DMA 各自成本；对 LoRA/ControlNet/参考图变化执行缓存失效检查。自身代码采用 LGPL，直接引入前逐文件核对许可。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
