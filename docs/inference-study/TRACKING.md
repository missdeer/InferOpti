# 研究范围与持续跟踪

## 研究记录：2026-11-26

目标：研究`inferences/`各项目面向消费级、小显存硬件的优化方法，积累ComfyUI后端经验。15个项目作为同级内容维护，各有项目笔记，共用一份版本、证据、文件和方法索引。

- 统一[snapshots.json](snapshots.json)保存15个项目、内嵌engine及ComfyUI基线的17条源码版本；每条保留原采集时间，父仓库观察另列`parent_history`。
- 统一[evidence.csv](evidence.csv)保存265条证据的源码位置与固定commit链接。
- 统一[source-files.csv](source-files.csv)保存22,181个文件的路径、大小和SHA-256；包括CUDA实际使用的`.inc`和`.glsl`片段，所有项目采用同一扩展名和排除规则。
- [METHODS.md](METHODS.md)按内存、搬运、计算、复用和调度归类；[COMFYUI.md](COMFYUI.md)按候选优先级、接入和验收组织，不按研究加入时间分章节。

历史采集数据的原commit与时间保留；旧文档组织可以从Git历史查阅。整理目录不代表重新核读或重新测量旧项目。

## 覆盖说明

本轮是围绕优化机制的静态研究，不是全量逐行审计。文件清单用于定位与差分，不代表每个文件都完成核读。

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
| tqLLM | dense CUDA格式/fit、TQ/TQT校准、KV分层、专家cache/prefill、Inkling驻留/spec、Metal调度/ANE、serve | 所有模型/vision/DSpine训练、全部内嵌shader、第三方代码及真机复测 |
| RustingStrata | Backend、expert cache/worker、CPU dot、prefill、CUDA/HIP/Vulkan队列/核资源、校准/CLI、代表性专用forward及状态缓存 | 全部模型/vision、HTTP/UI/MCP、量化表、CUDA/GLSL逐核与SPIR-V真机验证 |

`evidence.csv`是结论定位索引，项目文档和测试仅用于理解机制；有测试不表示本轮执行过或真卡已通过。

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

1. **先保存新版本，不覆盖旧快照。** 同一轮版本变更按日期保存所有受影响项目的历史快照；当前统一目录更新对应项目记录，比较每个 checkout HEAD 和父级 gitlink。若 dirty，保存文件差异范围，不能只记录 HEAD。
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

2026-11-26：统一档案检查通过15份项目笔记、17条源码版本、265条固定commit证据、22,181个文件SHA-256及523个本地链接。RustingStrata记录36项方法、42项证据和191个源码/测试/工具文件；工作区干净，HEAD与父仓库gitlink一致。旧项目元数据、证据及文件哈希完整保留，`.glsl`纳入统一源码清单。`git diff --check`通过。本轮未构建、未跑仓库单测、模型或硬件benchmark，结论仍是静态分析。
