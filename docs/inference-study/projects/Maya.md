# Maya：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`53885cef8faf0d3b3f7e28c0b1537a056c1f3ecf`
- 提交时间：`2026-11-25T21:45:25+00:00`
- 提交主题：README: donation addresses (Dash, Bitcoin, Ethereum) under the support buttons
- 源仓库：https://github.com/mw00/project-maya.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

继承 Strata 思路并围绕 GLM、DeepSeek、MiniMax 等模型做专用执行路径。重点不是把整张计算图搬到 C++，而是按硬件成本选择 CPU 专家计算、权重上传、GPU 计算和 SSD 读取。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 三层专家存储 | 热专家驻留 VRAM、其次在 RAM，剩余从文件读取；不同模型有独立实现。 | 不能因为某一个模型支持某路径就推断其他模型也支持。 | [src/core/expert_cache.cpp](../../../inferences/Maya/src/core/expert_cache.cpp)、[src/core/expert_source.cpp](../../../inferences/Maya/src/core/expert_source.cpp)、[src/ds4/ds4_model.cu](../../../inferences/Maya/src/ds4/ds4_model.cu) |
| 实测驱动的执行位置 | GLM 的 upload_rows_min_measured 比较一次上传与 CPU 固定/逐行成本；同一专家路由行数达到阈值才上传计算。 | 阈值随 PCIe、CPU、格式和负载变；不是固定“GPU 永远更快”。 | [include/maya/kernels/glmd_batch.hpp](../../../inferences/Maya/include/maya/kernels/glmd_batch.hpp)、[src/core/glm_dsa_prefill_stream_plan.cpp](../../../inferences/Maya/src/core/glm_dsa_prefill_stream_plan.cpp) |
| DeepSeek 路径校准与反馈 | calibrate_paths 测机器路径成本，prompt 路径选择对照预测耗时，后续用实际/预测修正。 | 这是具体模型内部调度；需要把校准开销与首次请求分开记录。 | [src/ds4/ds4_model.cu](../../../inferences/Maya/src/ds4/ds4_model.cu) |
| 热度衰减和淘汰 | HeatTable 维护衰减热度；从较旧的 LRU 窗口选低热 victim，设插入限额和收益 margin。 | MAYA_GLM_HEAT_EVICT 在 heat_on 中默认 false；不能把它写成全局默认。 | [src/core/glm_heat_evict.cpp](../../../inferences/Maya/src/core/glm_heat_evict.cpp) |
| 上传合并 | 相邻目标地址和源地址的 promotion 合并为更大 Run，完成后记录各对象的事件。 | 只合并满足同层/连续/可直接复制条件的条目。 | [src/core/glm_heat_evict.cpp](../../../inferences/Maya/src/core/glm_heat_evict.cpp) |
| NUMA 就近与分段锁页 | 按 NUMA 可用内存规划专家位置和 staging；大锁页区分段注册，失败时逐步缩小，保留成功前缀。 | 主要价值在多插槽/多卡；普通单插槽 PC 不能照搬复杂配置。 | [src/core/glm_tier_plan.cpp](../../../inferences/Maya/src/core/glm_tier_plan.cpp)、[src/core/glm_tier_numa.cu](../../../inferences/Maya/src/core/glm_tier_numa.cu) |
| 多卡预取重叠 | 每卡自己的层、链路和 double buffer；前一部分计算时预取后一部分的层。 | 真正重叠受 PCIe 拓扑、主存带宽与 staging 预算约束。 | [include/maya/kernels/glmd_batch.hpp](../../../inferences/Maya/include/maya/kernels/glmd_batch.hpp)、[src/core/glm_dsa_prefill_stream.cpp](../../../inferences/Maya/src/core/glm_dsa_prefill_stream.cpp) |
| Attention scratch 分块 | query 子批次×key block，不分配完整 T×max_context 分数；预算不足先缩 query，再缩 key block。 | 分块降低峰值；top-k/MLA/DSA 语义属于原模型，不能任意加到普通 DiT。 | [include/maya/kernels/glmd_batch.hpp](../../../inferences/Maya/include/maya/kernels/glmd_batch.hpp)、[src/core/glm_dsa_prompt_plan.cpp](../../../inferences/Maya/src/core/glm_dsa_prompt_plan.cpp) |
| Attention 遍历与 Tensor Core | Dense、DenseMasked、Gather 按有效上下文切换；WMMA attention 用在线 softmax/共享 tile。 | 必须保持相同选择集合和 mask；底层误差仍需对照。 | [include/maya/kernels/glmd_batch.hpp](../../../inferences/Maya/include/maya/kernels/glmd_batch.hpp)、[src/core/glm_prefill_dsa.cu](../../../inferences/Maya/src/core/glm_prefill_dsa.cu) |
| 图、投机和前缀缓存 | 层图、窗口验证、MTP/DFlash、n-gram proposal、prompt cache 都有单独模块。 | 接受率、回滚代价和 drafter 额外显存影响净收益。 | [src/core/glm_layer_graphs.hpp](../../../inferences/Maya/src/core/glm_layer_graphs.hpp)、[src/core/glm_dsa_spec.cpp](../../../inferences/Maya/src/core/glm_dsa_spec.cpp)、[src/core/glm_prompt_cache.cpp](../../../inferences/Maya/src/core/glm_prompt_cache.cpp) |
| 模型专用格式及量化包 | GGUF/EXL3 pack 转换、原生专家布局和量化包选择减少通用框架开销。 | 仓库有 pack/校准工具，不代表所有已发布量化包的训练与优化过程都在本地完整公开。 | [tools/iq_pack.py](../../../inferences/Maya/tools/iq_pack.py)、[tools/exl3_pack.py](../../../inferences/Maya/tools/exl3_pack.py)、[tools/maya_quants.py](../../../inferences/Maya/tools/maya_quants.py) |

## 对 ComfyUI 的借鉴

最有用的是“按测得成本决定上传还是原地算”、Attention 内存上界、权重拷贝合并与跨阶段预取。将策略迁移为少量可解释的成本规则，比移植一整套模型专属巨型执行器更现实。

## 下次跟踪

跟踪不同模型的能力矩阵；为 CPU fallback、热淘汰默认值、路径校准与量化包分别建实验。DSA 的选择算法和通用 attention kernel 优化要分开。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
