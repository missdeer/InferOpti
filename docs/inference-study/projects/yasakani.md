# yasakani：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`631fdfd01f071bdb9131e26da8e06d5ddc39b2a8`
- 提交时间：`2026-11-25T13:58:41+09:00`
- 提交主题：docs: the names of the two users who reported the 0.3.4 rule on 24 GB cards (0.3.5 entry)
- 源仓库：https://github.com/yasakani-dev/yasakani.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

专用压缩格式与 CUDA kernel 协同设计的引擎，低位权重、KV 压缩、copy/edit proposal、宽验证 pass 是主要策略。其极高编辑吞吐与输入复用高度相关，不是通用图像推理加速倍数。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 带旋转的压缩权重格式 | YSKN tensor 记录 trellis/LUT/scale/stream 与输入输出旋转描述，运行时按其 layout 读取。 | 私有容器及预处理不是标准 INT4；相对原始模型存在量化质量差异。 | [src/format.hpp](../../../inferences/yasakani/src/format.hpp)、[src/gemm_compressed.cu](../../../inferences/yasakani/src/gemm_compressed.cu) |
| 压缩权重直接 MMA | gemm_compressed 将 codebook/压缩 fragments 在共享内存/寄存器中解码为 int8，执行 MMA；cp.async 双级 pipeline。 | 避免全模型展开，但额外 lookup、scale、部分和也要计成本。 | [src/gemm_compressed.cu](../../../inferences/yasakani/src/gemm_compressed.cu) |
| 宽 pass 与稳定求和 | wide/exact-sum GEMV 将更多 proposal 一次验证，固定数值求和约束以接近或匹配小 pass。 | engine35 对宽度/能力和 xsum eligibility 作条件检查；更宽需要额外 workspace。 | [src/engine35.cpp](../../../inferences/yasakani/src/engine35.cpp) |
| copy-from-input 索引 | CopyIndex 维护四 token window 哈希链，选顺序续拷或最长 backward match 作为 draft。 | 只产生 proposal，target 仍验证；对非复制型输出接受率较低。 | [src/copy_index.hpp](../../../inferences/yasakani/src/copy_index.hpp) |
| 重复编辑规则学习 | EditCopy 在两个不同输入位置得到同替换证据才提出规则；根据 unchanged/miss 计数停用不划算规则。 | 源码明确由调用方 target verification 保证正确；不是跳过模型直接替换字符串。 | [src/edit_copy.hpp](../../../inferences/yasakani/src/edit_copy.hpp) |
| KV 低位与旋转 codec | KV 压缩/旋转与 fused attention 路径配套，减少长上下文容量与读带宽。 | 不同 KV mode 的误差不同；不能把相对压缩模型的 exact pass 称为 BF16 无损。 | [src/kvx_codec.cuh](../../../inferences/yasakani/src/kvx_codec.cuh)、[src/attention_kv4q.cu](../../../inferences/yasakani/src/attention_kv4q.cu)、[src/kvq_attn.cuh](../../../inferences/yasakani/src/kvq_attn.cuh) |
| Prefill v2 与上下文快照 | 新 prompt 路径调整布局、共享行压缩、输出 head 计算及快照；旧模式保留对照。 | 更大 workspace 可能不适合小卡；Python 入口和 C++ 环境开关的默认不是同一层。 | [src/prefill_mode.hpp](../../../inferences/yasakani/src/prefill_mode.hpp)、[src/q35_prefill.cu](../../../inferences/yasakani/src/q35_prefill.cu)、[src/q35_chk.cu](../../../inferences/yasakani/src/q35_chk.cu) |
| 图与多请求状态 | engine35 缓存不同 pass、MTP、多 slot 图并清理失效图；每个 stream 有自己的采样状态。 | 共享 pass 的形状和状态需验收；多请求并不总比单请求顺序编辑快。 | [src/engine35.cpp](../../../inferences/yasakani/src/engine35.cpp)、[README.md](../../../inferences/yasakani/README.md) |
| 按真实编译代码检查能力 | select_gpu 同时检查设备 compute capability 与探针 kernel 的 ptxVersion；PDL/bulk prefetch 仅在实际代码可用时开。 | 新 GPU 上运行旧架构编译码，不代表自动具备新架构指令。 | [src/gpu_caps.cu](../../../inferences/yasakani/src/gpu_caps.cu) |
| 质量与 benchmark 分项披露 | README 区分量化质量、首次/热 prompt、复制任务、并发限制及自检范围。 | 项目数字条件不统一，不能把编辑 tok/s 与 diffusion steps/s 或图片/秒换算。 | [README.md](../../../inferences/yasakani/README.md) |

## 对 ComfyUI 的借鉴

值得借鉴压缩布局内算、启动能力检查、宽度不变性测试，以及拒绝低收益 proposal 的原则。复制 draft/编辑规则仅能帮助 ComfyUI 内的文本模型环节，对 DiT 无直接替换。

## 下次跟踪

追踪较小卡的 workspace fallback、KV 质量、宽 pass 的数值自检和 cold/repeated 分类；所有推测性加速要报告 target calls、proposal cost 和接受长度。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
