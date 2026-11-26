# Speedy：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`1f172963a0a483672381e74254b283df22c25e81`
- 提交时间：`2026-11-24T07:53:23+02:00`
- 提交主题：Speedy 0.5.5 (#32)
- 源仓库：https://github.com/DevDock-AI/Speedy.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

Strata 的测量与配置导向分支，既有推理路径调整，也有面向 agent 的服务端行为调整。后一类不应混入 GPU 算子提速统计。不同版本的 Strata 之间有目录和时间差，文件差异不自动等于 Speedy 独创。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 按硬件验证的默认组合 | speedy_defaults 给未设置的环境变量填一组 production defaults，保留用户 override 与 upstream defaults 退出。 | 注释测量基准是特定 RTX 3080 配置；不能假定适合任意 8GB/12GB 显卡。 | [src/program/speedy_defaults.hpp](../../../inferences/Speedy/src/program/speedy_defaults.hpp) |
| 显式关闭有损路由省略 | 默认将 ROUTE_TAIL_SKIP=0；将数值质量和执行加速分开。 | 这只是一个开关的默认，不保证所有路径跨后端逐位相同。 | [src/program/speedy_defaults.hpp](../../../inferences/Speedy/src/program/speedy_defaults.hpp) |
| CPU 等待及传输策略 | 配置 PIN_BY_NEED、VERIFY_DEFER_CPU 及阈值、EXCHANGE_ROTATE，协调 CPU miss 与 GPU 路径。 | 是否触发要看 verify.cpp 的分支和真实命中率。 | [src/program/speedy_defaults.hpp](../../../inferences/Speedy/src/program/speedy_defaults.hpp)、[src/core/verify.cpp](../../../inferences/Speedy/src/core/verify.cpp) |
| Prefill 工作区与执行区域 | 开启 PF_GROUP_SCRATCH、PF_HC_REGION，调整大块 prompt 的 scratch/区域执行。 | 属于负载和显存相关调优；注释中数字是项目记录，不是本次复测。 | [src/program/speedy_defaults.hpp](../../../inferences/Speedy/src/program/speedy_defaults.hpp)、[src/prefill/prefill.cpp](../../../inferences/Speedy/src/prefill/prefill.cpp) |
| 量化 MMQ 保持压缩布局计算 | MoE prompt 对 K-quant/Gyro 等分派量化矩阵乘，避免通用 FP16 展开路径的工作区。 | 编译和运行开关共同控制，支持格式表决定实际命中。 | [src/prefill/moe_mmq.cu](../../../inferences/Speedy/src/prefill/moe_mmq.cu) |
| 可选核模块及回退 | 独立 K-quant DSO 在 cache 定尺寸前加载，核对 ABI 与参数大小；缺失/失败回到 FP16。 | 模块装载本身占资源，必须在剩余显存规划前做。 | [src/prefill/mmq_kq_loader.cpp](../../../inferences/Speedy/src/prefill/mmq_kq_loader.cpp) |
| Attention 和 KV 预取 | 生产默认启用 PROMPT_ATTN_IMMA 与 KV_PREFETCH，配合已有量化 KV/attention 路径。 | 不能只从默认值推断端到端收益，应测 attention 占比和 cache 压力。 | [src/program/speedy_defaults.hpp](../../../inferences/Speedy/src/program/speedy_defaults.hpp)、[src/core/layer.cpp](../../../inferences/Speedy/src/core/layer.cpp) |
| 投机与前缀复用 | 延续 suffix drafter、draft policy、会话缓存，配套服务端请求生命周期。 | agent 防循环/工具调用修复是成功率与重试成本优化，不是纯 kernel 优化。 | [src/spec/draft_policy.cpp](../../../inferences/Speedy/src/spec/draft_policy.cpp)、[src/spec/suffix_drafter.cpp](../../../inferences/Speedy/src/spec/suffix_drafter.cpp)、[src/core/conversation_state.cpp](../../../inferences/Speedy/src/core/conversation_state.cpp) |

## 对 ComfyUI 的借鉴

最值得学的是固定硬件上先做好基线和默认组合，再决定是否写新 kernel。原生新增贡献标为 Personal & Non-Commercial，商业代码复用不能直接按 Strata 的 MIT 推定。

## 下次跟踪

更新时分别核对 engine diff、默认值 diff、服务端行为 diff；在同一版本和同一量化包上做比较。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
