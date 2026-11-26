# 消费级硬件推理引擎源码学习档案

本档案服务于 ComfyUI 推理后端优化。研究日期为 **2026-11-26**，目前覆盖本机 `inferences/` 中 **14 个项目**的固定 checkout：初次13个项目，另以独立增量加入tqLLM。没有拉取远端更新，没有构建或运行模型。源码存在、入口启用、真机验证和可迁移收益是四件不同的事。

主要结论：这批项目的共同价值包括**权重与状态分层、显存和锁页预算、数据搬运与计算重叠、压缩布局与算子协同、跨请求复用、按硬件成本调度、严格数值验证**。用“节点间异步 + 单节点新算法”概括会漏掉资源规划和重复工作消除。

最直接的图像参考是 **XeStrata 的 LayerStream/DiT/VAE**、**TensorFold 的图像 family**，其次是 Strata 的原生图像实验。Maya 的 CPU/GPU 成本选择与三级存储经验很有价值，但大量机制依赖 MoE，不能照搬到 dense DiT。**已有 ComfyUI 本地基线也包含预取和图捕获，后续要比较差异和净收益。**

## 阅读入口

| 文档 | 用途 |
| --- | --- |
| [版本快照](SNAPSHOTS.md) | 每个项目完整 Git commit、嵌套子模块、基线与定位方式 |
| [跨项目方法索引](METHODS.md) | 按问题检索方法，比较可迁移性和代价 |
| [ComfyUI 迁移与实验清单](COMFYUI.md) | 对照当前实现，确定候选改进与验收要求 |
| [研究范围与跟踪记录](TRACKING.md) | 已覆盖路径、未核读范围、后续更新流程 |
| [snapshots.json](snapshots.json) | 机器可读的 HEAD、日期、主题、分支、remote、工作区状态及 gitlink |
| [evidence.csv](evidence.csv) | 162 个源码/说明定位条目，附固定 commit 的远端链接 |
| [source-files.csv](source-files.csv) | 21,088 个纳入清单的源码、测试和工具文件及 SHA-256 |
| [inventory-summary.json](inventory-summary.json) | 扩展名、排除规则及项目文件计数 |
| [tqLLM 增量归档](updates/2026-11-26-tqLLM/README.md) | 新增项目完整commit、61项证据、825个文件哈希；原始清单保持原样 |

初次归档的数字和机器可读文件保留为历史。加上tqLLM后累计223项证据、21,913个文件清单条目；本次新增`.inc`范围，计数口径见增量说明。

## 逐项目记录

| 项目 | 本轮重点 | 方法笔记 |
| --- | --- | --- |
| Strata | 专家缓存、CPU 池、图、投机、pinned 预算、图像实验 | [Strata](projects/Strata.md) |
| Maya | 按实测成本分工、NUMA、DSA 分块、分层流式执行 | [Maya](projects/Maya.md) |
| XeStrata | 原生 SYCL、DiT 层流式预取、条件缓存、W8A8、VAE tile | [XeStrata](projects/XeStrata.md) |
| XeMaya | XeStrata 基础上的模型适配、直接读取、批量装填、prompt 保留 | [XeMaya](projects/XeMaya.md) |
| Speedy | 固定硬件默认组合、MMQ、CPU/GPU 等待、工程验证 | [Speedy](projects/Speedy.md) |
| Sync-Strata | trellis 量化包、codebook、混合率、CPU ISA 与 GPU 算子 | [sync-strata](projects/sync-strata.md) |
| Basalt | Blackwell 格式/算子协同、persistent kernel、peer tier | [basalt](projects/basalt.md) |
| QuixiInfer | RAM/NVMe tier、io_uring、KV 页、显存借还、连续服务 | [QuixiInfer](projects/QuixiInfer.md) |
| Cheetah | mmap 与页控制、native CPU、Q8 KV、Vulkan 能力边界 | [Cheetah](projects/Cheetah.md) |
| Kinetic | 前缀 exact state、共享 pass、树形投机、专家分池 | [kinetic](projects/kinetic.md) |
| Splash | Apple 统一内存治理、Metal 调优、ANE/GPU 分工 | [splash](projects/splash.md) |
| TensorFold | 内存准入、arena、图缓存、Qwen/MOSS 图像与窄范围 ComfyUI 接入 | [TensorFold](projects/TensorFold.md) |
| YASAKANI | 压缩计算、KV codec、copy/edit draft、宽 pass 与能力检查 | [yasakani](projects/yasakani.md) |
| tqLLM | TQ/TQT校准与native GEMM、成本交点、自动fit、KV分层、驻留重规划、精度契约 | [tqLLM](projects/tqLLM.md) |

## 如何理解覆盖程度

已对每个项目的目录、说明、推理实现入口和相关核心源文件做静态阅读，沿内存、计算、缓存和调度路径记录方法。**这不是所有清单文件的逐行全量审计**；文件清单用于定位和后续差分，不代表逐个文件已核读。详细取舍与仍需深挖的部分见 [TRACKING.md](TRACKING.md)。没有把 README 的性能数字当成本次测量，也没有判定这些项目在目标消费卡上的速度排名。

主线关系应理解为代码与思路的继承：Strata → Speedy / Sync-Strata / Basalt；Maya 有 Strata 血缘并发展模型专用执行器；XeStrata 源自 Strata；XeMaya 的直接基础是 XeStrata，参考 Maya 的模型与量化方案。**未根据目录名称假定所有分支都从同一个版本分叉，也未声称所有相同方法都是各项目独立发明。**

原技术路线文档见 [ComfyUI-Optimization-tech.md](../../ComfyUI-Optimization-tech.md)。本档案保留源码层面的事实、边界和待验证项，后续实测结果应作为独立记录追加。
