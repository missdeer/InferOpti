# 研究范围与持续跟踪

## 2026-11-26 初次归档

用户目标：仔细研究 `inferences/` 各项目针对消费级、小显存硬件的优化机制，持续积累能够用于 ComfyUI 后端的经验；逐项目持久化策略，并保存当前完整 Git commit。

已完成：

- 清点 13 个项目及目录布局，读取相关 AGENTS 和项目说明。
- 固定每个项目完整 HEAD、提交日期/主题、remote、branch、工作区与父级 gitlink；额外固定 Sync-Strata engine 和本地 ComfyUI 基线。
- 对每个项目追踪核心内存、计算、缓存、调度实现，生成 13 份独立方法笔记和 162 项源码/说明证据定位。
- 建立跨项目方法分类和 ComfyUI 迁移/实验清单；区分 MoE 与 dense DiT、默认与 opt-in、容量与速度、项目报告与本次验证。
- 为 21,088 个纳入清单的源码/测试/工具文件保存路径、字节数、SHA-256，便于以后差分。

## 覆盖说明

本轮是**围绕优化机制的静态研究**，不是全量代码审计，也没有逐行读完所有文件。主要沿以下调用链阅读：

| 项目组 | 核心阅读路径 | 未展开的主要内容 |
| --- | --- | --- |
| Strata / Speedy / Sync-Strata | expert cache/source、CPU pool、layer/verify、prefill、graph、spec controller、TQ/MMQ、图像入口 | app/terminal 所有交互、全部模型和后端的逐算子证明、所有 benchmark fixture |
| Maya | GLM tier/heat/NUMA、DSA prompt plans、DeepSeek cost calibration/path selection、pack | 每个专用模型的全部 kernel、量化包训练全过程、Metal/HIP 全能力矩阵 |
| XeStrata | LayerStream、DiT/产品/行操作、预算、TextEncoder、VAE tile、Flash Attention、设备分派 | 所有图像 family、训练、辅助视觉网络、各设备 ISA 及完整服务端 |
| XeMaya | PlainModel、read_blobs、fill_slots、prefill、STATUS/PLAN | 各模型完整 forward、所有真实硬件报告复现、继承图像路径的运行验收 |
| Basalt | resident plan、prefill streaming、window NVFP4、persistent dense kernel、CPU fallback、验证规范 | 所有模型与 drafter 的完整训练、全部专用 kernel 的 SASS/性能分析 |
| QuixiInfer | ExpertStore、HostExperts、KV pages、driver/mailbox、内存借还、allocator | 所有 GPU kernel/grammar/vision、真实长上下文和多槽压力复现 |
| Cheetah | engine、mmap store、native ops/kernel/decoder、Vulkan Q4 shader、prefix | 全部 family registration 与 API/MCP；Vulkan 分层的完整真机内存验证 |
| Kinetic | placement plan、Warm、Convoy、spec policy/tree、CUDA graph/attention、Folds | 所有 model block、全部 CPU/Vulkan/CUDA kernel、已披露问题是否在其他版本修复 |
| Splash | MemoryPlan/Governor、WeightResidency、Scheduler、KernelSelection、ANE calibration | 每种 Metal kernel、全部模型 streaming/cache 细节、ANE 真机覆盖 |
| TensorFold | CUDA 内存/arena/graph、Metal resident、Qwen/MOSS 图像、ComfyUI installer | 数千其他 family/test/kernel 文件、语音/训练/app/cluster；没有下载 pinned 媒体历史源包 |
| YASAKANI | 压缩 GEMM、format、GPU caps、Engine35、copy/edit proposal、KV codec、prefill 模式 | 全部私有格式转换训练、每种 attention/codec 模式、真机宽 pass 自检 |

`evidence.csv` 是结论引用的位置索引，也不是对每个文件完整核读的声明。仓库内文档、注释和测试代码用于理解机制；“有测试”不表示本次执行过或真卡已通过。

## 证据状态

后续更新每条方法时保留四个维度：

| 维度 | 可用状态 | 记录要求 |
| --- | --- | --- |
| 实现 | 源码可见 / 仅文档 / 未找到 | 文件、关键符号、完整 commit |
| 入口 | 默认 / opt-in / 测试入口 / 不确定 | CLI、配置或环境开关及调用条件 |
| 验证 | 本轮静态 / 项目自报 / 本地复测 | 不把项目自报转写成本地实测 |
| 迁移 | 可直接对照 / 条件适用 / 不直接适用 | 说明是机制借鉴、代码复用还是新实验 |

本轮没有编译、运行测试或模型，没有调整 GPU/电源/系统设置，没有安装依赖、下载权重、联网核验远端，也没有改动任何推理项目或 ComfyUI 的源码。文档中的收益机制与迁移判断是静态分析；性能和画质结论仍待实测。

## 下一轮更新流程

1. **先保存新版本，不覆盖旧快照。** 将新的 snapshots 放到带日期的目录；比较每个 checkout HEAD 和父级 gitlink。若 dirty，保存文件差异范围，不能只记录 HEAD。
2. **按路径定位相关变化。** 先比较上述核心文件；目录移动时用 commit、符号和内容 hash 定位，不依赖旧路径名。
3. **继承和新增分开。** 相同方法可能从上游同步，不能重复记作新发明；不同年份/版本的文件差异也不一定是分支独有改动。
4. **确认开关及调用链。** 代码仍存在但入口禁用、默认改变、模型未适配、功能只进测试入口时，更新状态。
5. **只对变化的机制补读。** 单独维护待深挖项，避免把模型/GUI 新功能和后端性能混在一起。
6. **有条件再做真机 A/B。** 按 COMFYUI.md 的分阶段、冷热、质量与失败要求记录；保留负面结果。

建议每次实验记录以下字段，尚未测量时留空，不填估计成绩：

```text
日期 / 项目 / 源 commit / 模型与权重 hash / 后端与驱动
硬件 / VRAM 与 RAM 上限 / 工作流与输入 hash
方法 / 开关 / 对照 / 数值语义是否变化
冷启动 / 首次 / 热缓存 / 单请求 / 并发
墙钟 / GPU 时间 / 传输字节 / 显存 / RAM-pinned / 质量 / 失败
结论：保留、拒绝、噪声内、待验证；原因和复测条件
```

源码清单排除 `third_party`、`node_modules`、`ref`、fixtures/results、工具缓存等路径；纳入扩展名和精确排除表在 inventory-summary.json。它用于研究跟踪，不是构建依赖的完整可复现包，也不保证远端永久保留 commit。

## 归档一致性检查

2026-11-26：检查通过 18 份档案 Markdown 的 313 个本地链接、16 个 Git HEAD 快照、162 个证据条目及 21,088 个文件 SHA-256；13 个推理项目和嵌套 engine 仍为干净工作区，父级 gitlink 与所读 HEAD 一致。各项目笔记中的完整 commit 与快照一致。这些检查只验证文档定位和源码版本一致性，不验证推理功能、数值、速度或画质。

## 2026-11-26：增量学习 tqLLM

新增第14个项目的[学习笔记](projects/tqLLM.md)，38项方法、61个源码/说明证据条目。HEAD为`7df840632cd3f108de459e0944b9b6d115532689`，1.8.3；tracked clean、untracked 0；与父仓库本次索引gitlink相同。父仓库HEAD为`adfa2ae9a4c04fedbb66cce5a9bbff82f3c635bb`。

本次元数据单独存到[updates/2026-11-26-tqLLM](updates/2026-11-26-tqLLM/README.md)，保留初次JSON/CSV。新增825个源/测试/工具文件哈希，包含实际CUDA include使用的`.inc`，归档旧源码单独分类。累计14个项目、223项证据、21,913条文件清单，两个批次扩展名范围不同。

核心阅读覆盖：dense CUDA格式/加载/forward/fit、TQ/TQT布局及离线GPTQ/Viterbi、KV store/codec/compact、Flash-Next专家cache与prefill、Inkling驻留重规划/投机、Metal宽度/能力/融合/ANE、serve scheduler/router。没有逐行核读全部模型/vision/DSpine训练、全部内嵌shader或第三方代码，也未运行本项目测试/模型/benchmark。

特别记录：fit_lossy与exact的边界；低位存储与实际MMA不同；首次draft路径也需重规划；数值不同kernel不由启动测速噪声选择；历史测试注释可能落后于当前native GEMM能力。新增ComfyUI候选仍为待验证。

本次归档检查通过：14份项目笔记、418个本地链接（含根README）、38项tqLLM方法、61个固定commit证据、825个新增文件SHA-256，tqLLM HEAD/干净工作区/gitlink一致，初次机器可读档案计数保持不变。`git diff --check`通过；未运行推理代码或硬件测试。
