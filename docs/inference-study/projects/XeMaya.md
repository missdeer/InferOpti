# XeMaya：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`8fa4141666e10d81bf932ab4625aa52a37eed4bd`
- 提交时间：`2026-11-12T04:40:11+00:00`
- 提交主题：xemaya: 0.1.0-beta.1 published; what its build was checked on
- 源仓库：https://github.com/Jumbomuffin777/XeMaya.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

直接代码基础是 XeStrata，不是把 Maya 全量 CUDA 自动翻译成 SYCL。xemaya/PLAN.md 明确 Maya 主要提供量化包和思路；PlainModel 与原生 SYCL kernel 是实现基础。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 继承的原生 SYCL 计算 | XMX/joint_matrix、DP4a/FMA fallback、CPU AVX 专家及分层存储沿 XeStrata 架构。 | 继承项不能重复计作新算法贡献。 | [xemaya/PLAN.md](../../../inferences/XeMaya/xemaya/PLAN.md)、[src/core/plain_model.cpp](../../../inferences/XeMaya/src/core/plain_model.cpp) |
| 文件直接读入 pinned staging | FileExpertSource.read_blobs 多线程将 transient 专家读进调用方 staging，省去额外中转 memcpy。 | 仅能直接读取的条目走此路，其余 copy_blob 回退。 | [src/core/expert_source.cpp](../../../inferences/XeMaya/src/core/expert_source.cpp)、[src/prefill/prefill.cpp](../../../inferences/XeMaya/src/prefill/prefill.cpp) |
| 批量填 GPU 槽 | fill_slots 收集 layer/expert/dst，read_blobs 并行准备，fill_slot_queued 入队后统一同步再发布 residency。 | 必须在 DMA 完成后公布映射；不是先写 slot 表再搬权重。 | [src/core/plain_model.cpp](../../../inferences/XeMaya/src/core/plain_model.cpp) |
| Prefill 期间保留热点 | prompt_routed 统计路由，空闲 GPU 槽保存 prompt 热专家，后续 chunk 不必再次读取。 | STATUS 说明默认开启，PROMPT_KEEP=0 可关闭；实际数值依模型与机器。 | [src/core/plain_model.cpp](../../../inferences/XeMaya/src/core/plain_model.cpp)、[xemaya/STATUS.md](../../../inferences/XeMaya/xemaya/STATUS.md) |
| 可选绕过页缓存 | unbuffered_read 支持不经 OS page cache 的读取。 | EXPERT_UNBUFFERED=1 为 opt-in；项目记录显示 prefill 可快但后续 CPU 专家 decode 会因 cache 冷而慢。 | [src/platform/unbuffered_read.cpp](../../../inferences/XeMaya/src/platform/unbuffered_read.cpp)、[xemaya/STATUS.md](../../../inferences/XeMaya/xemaya/STATUS.md) |
| 多卡按层切分 | PlainModel chain 在不同设备维护各自状态、缓存和执行，结合本地专家池。 | 额外卡的 PCIe 位置可能拖累整机；不能只加总显存。 | [src/core/plain_model.cpp](../../../inferences/XeMaya/src/core/plain_model.cpp)、[xemaya/AGENTS.md](../../../inferences/XeMaya/xemaya/AGENTS.md) |
| 状态回滚与学习后 profile | rollback 恢复递归 state/conv；save_profile 汇总 chain 各层的常驻和热度。 | 保证投机失败不污染后续状态；profile 必须与模型版本匹配。 | [src/core/plain_model.cpp](../../../inferences/XeMaya/src/core/plain_model.cpp) |
| 参照输出验证与限制记录 | parity 与 CPU reference、真实模型短序列比较；STATUS 区分 CPU 校验、真卡校验和未验证平台。 | 本次未重跑项目报告的数字；CPU device parity 不能代替 XMX 真卡验收。 | [xemaya/STATUS.md](../../../inferences/XeMaya/xemaya/STATUS.md)、[xemaya/PLAN.md](../../../inferences/XeMaya/xemaya/PLAN.md) |

## 对 ComfyUI 的借鉴

借鉴最小拷贝路径、批量装填和“先完成后发布”的缓存协议。其负面经验同样有价值：绕过 page cache 可能牺牲后续阶段；多卡平台要检查共享 chipset/NIC 链路。

## 下次跟踪

GLM/DeepSeek/MiniMax 与多卡、drafter、vision 的完成度不同；按 STATUS 的具体检查范围追踪。不要将继承的 image 目录直接认定为 XeMaya 当前已验收的图像产品。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
