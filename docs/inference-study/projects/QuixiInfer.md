# QuixiInfer：消费级硬件优化方法

研究日期：2026-11-26。证据范围：本地 checkout 的静态源码与项目记录；本次未构建、未运行模型、未复现 benchmark。

- 完整 commit：`dc5dbc1bb1cf7a31f6e8a240f7a34bdf2014268b`
- 提交时间：`2026-11-20T19:13:34-05:00`
- 提交主题：handoff and README: QX_DS_CHUNK_RES=1e6 is on in the production unit
- 源仓库：https://github.com/QuixiAI/QuixiInfer.git
- 源码状态：tracked clean，untracked 0；总表见 [版本快照](../SNAPSHOTS.md)。

独立 Rust/CUDA 引擎，面向单工作站承载大 MoE 与多个请求；以服务容量、调度一致性和 RAM/NVMe 层级为重点。其多槽和显存借用设计可供 ComfyUI 服务化参考。

## 方法与策略

下表“已实现”仅指找到对应代码，不代表当前入口必然启用或已在本机验证。默认/可选/实验条件以该行说明为准。

| 方法 | 源码机制与目的 | 执行条件、边界与风险 | 主要源码 |
| --- | --- | --- | --- |
| 三级专家存储 | 按相同 record size 建 class，VRAM 外的记录保存在 pinned RAM；剩余按路由从 NVMe 读取。 | 面向大型 MoE；读取了多少模型数据需和 tok/s 一起记录。 | [crates/qx-engine/src/tiers.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/tiers.rs) |
| O_DIRECT 与 io_uring | 4KiB 对齐 record 直接读入 pinned place，最多 64 在途读取，promotion 用独立 ring。 | Linux 定向实现；不是 Windows 直接可复用的接口。 | [crates/qx-engine/src/tiers.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/tiers.rs) |
| Host 布局与大页 | gate/up/down 打包连续专家记录，匿名 mmap + MADV_HUGEPAGE + driver register；pack 可直接填入。 | 减少 TLB 与多次 memcpy，也避免文件 page cache 额外副本；需付 pinned 预算。 | [crates/qx-engine/src/host_experts.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/host_experts.rs) |
| 热点升降级与安全隔离 | driver token 间换热专家，释放的槽隔离两个 token 防止在途工作读到被覆盖数据。 | 必须把异步读者生命周期纳入缓存替换。 | [crates/qx-engine/src/dsv41/driver.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/dsv41/driver.rs) |
| RAM 弹性归还 | cold records 位于独立 elastic arena，释放时回落 NVMe，恢复后重新填。 | 容量可退让但期间产生更多 NVMe miss；这是吞吐/容量交换。 | [crates/qx-engine/src/tiers.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/tiers.rs) |
| 空闲显存借给专家 | driver 有 flex/lend 请求与 ack，把暂未使用的 slot 尾部让给 tier。 | 使用前必须收回，不能让 KV 与专家同时拥有同一物理空间。 | [crates/qx-engine/src/dsv41/driver.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/dsv41/driver.rs) |
| 分页 FP8 KV 和共享前缀 | 256-cell page 包含 K/V 与分组 scale，引用计数共享页；host KV 和 GPU pooled-key 页分别寻址。 | 按实际使用上下文付费，仍须统计整块 arena 预留与真实提交。 | [crates/qx-engine/src/kvpages.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/kvpages.rs) |
| 异构执行与预测观测 | mailbox 通知 CPU miss；driver 统计 pool、staging、mapped RAM、prefetch 命中与耗时。 | 字段存在不表示每个实验开关默认启用；zero-copy 不保证快。 | [crates/qx-engine/src/dsv41/driver.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/dsv41/driver.rs) |
| 主机 allocator 调整 | 启动前固定 glibc mmap/trim 阈值，减少多线程中型临时块滞留 arena 造成的 RSS 增长。 | Linux glibc 条件分支；项目记录的服务负载现象不能推广为所有应用默认值。 | [crates/qx-engine/src/lib.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/lib.rs) |
| 连续批处理与状态复用 | 多 slot 请求、prefix、park/resume、投机都有服务/engine 模块，并有 batch/invariance 对照工具。 | 调度后结果要与 solo 一致；LLM slot 状态不能直接当作扩散 latent 状态。 | [crates/qx-engine/src/dsv41/mod.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/dsv41/mod.rs)、[crates/qx-engine/src/prefix.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/prefix.rs)、[crates/qx-engine/src/bin/qx-invariance.rs](../../../inferences/QuixiInfer/crates/qx-engine/src/bin/qx-invariance.rs) |

## 对 ComfyUI 的借鉴

借鉴临时工作区的借还协议、按大小分池、热点升级的隔离期、服务内存增长诊断和请求准入。对 dense diffusion，从 SSD 反复读权重通常应作为容量兜底而非提速目标。

## 下次跟踪

追踪多请求 p95、每请求/聚合吞吐、NVMe GB/输出和 host RSS；不要只比较 README 单流数字。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI 迁移与实验清单](../COMFYUI.md)
- [源码定位与固定版本链接](../evidence.csv)
