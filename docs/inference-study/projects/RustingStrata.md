# RustingStrata：分层专家、跨后端执行与消费卡资源规划

研究日期：2026-11-26。本轮为静态源码研究，未构建、未运行模型或重测benchmark。

- 完整commit：`e50ec2cc0ecf802319623685ef41d2101d19561b`
- 提交时间：`2026-11-26T09:45:19+01:00`；原始采集UTC：`2026-11-26T09:36:12.272371+00:00`。
- 提交主题：raon: the greedy picks use ops::argmax (no panic on NaN logits); a drafter block size under 2 no longer panics in clamp; a zero embedding, head, rank or expert count in the metadata is an error; expert_weights_scale 0 means no scaling, as in llama.cpp
- 本地分支：`main`；tracked clean、untracked 0；Cargo/README版本1.3.7。
- 源仓库：https://github.com/RustingStudio/RustingStrata.git
- [统一版本快照](../SNAPSHOTS.md)、[源码证据](../evidence.csv)、[文件哈希](../source-files.csv)。

## 核心结论与关系

RustingStrata围绕超出VRAM的MoE设计：**GPU放热点专家、CPU算miss、mailbox汇合；prefill按专家分组并流水搬运；图和状态复用降低主机开销。** 它还提供运行时编译缓存、多后端资源预检、带预算的参数校准以及slot准入。

不能仅凭名字认定它直接继承完整Strata执行器。当前源码是Rust的Backend与模型路径、手写CUDA device源、Vulkan GLSL/SPIR-V，CPU dot等注明参考llama.cpp；THANKS明确HIP compat借鉴Strata（MIT）。CUDA/HIP共享设备源码，Vulkan是另一实现并努力对齐数值。“pure Rust”主要描述host/build体验，GPU kernel不是Rust源码，运行仍依赖驱动和NVRTC/hiprtc等库。

本轮没有发现DiT/VAE图像生成后端；vision目录是模型图像输入编码，不能当作ComfyUI图像生成替代。直接价值是资源规划、生命周期、预取、调优和异常恢复经验。

## 方法与策略

源码可见不等于每个入口默认启用。默认/opt-in、模型范围和后端差异列在各行；项目自报速度不视为本轮实测。

| 方法 | 源码机制与目的 | 执行条件、默认与边界 | 主要源码 |
| --- | --- | --- | --- |
| 独立Rust控制层、多后端契约 | Backend定义buffer、graph、batch、beside及计算；CUDA/HIP共享kernels.cu的可移植分支，Vulkan另有GLSL/SPIR-V实现。 | 不是完整Strata C++引擎的Rust包装，也不是原CUDA文本直接作为Vulkan shader；THANKS明确部分HIP兼容代码参考Strata。 | [src/backend.rs](../../../inferences/RustingStrata/src/backend.rs)、[src/hip/mod.rs](../../../inferences/RustingStrata/src/hip/mod.rs)、[THANKS.md](../../../inferences/RustingStrata/THANKS.md) |
| 运行时编译和机器码磁盘缓存 | CUDA用NVRTC生成cubin，key含生成源码、目标架构、NVRTC版本；HIP用hiprtc为gfx目标编译。 | 首轮编译与加载缓存分开计时；CUDA缺NVRTC/初始化条件可回落Vulkan，不是任意CUDA运行错误都静默回落。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/cuda/nvrtc.rs](../../../inferences/RustingStrata/src/cuda/nvrtc.rs)、[src/backend.rs](../../../inferences/RustingStrata/src/backend.rs) |
| 按设备能力保留旧卡路径 | sm70/75和sm80+分派FP16 Tensor Core/FMA、INT8可用性、cp.async或同步拷贝；不支持的架构明确拒绝。 | NVRTC13不支持Volta，需12.x；有Tensor Core不代表有当前INT8 MMA。HIP为实验后端，不能由CUDA性能推导。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/cuda/kernels.cu](../../../inferences/RustingStrata/src/cuda/kernels.cu)、[src/hip/mod.rs](../../../inferences/RustingStrata/src/hip/mod.rs) |
| Vulkan矩阵能力和子组分派 | 检测cooperative matrix FP16/INT8形状，缺失时用packed dot/tile路径；支持逻辑wave32或固定wave64。 | Vulkan入口要求shaderFloat64和整数/子组能力，README说明Intel Arc缺该能力；不是全部Vulkan GPU均能运行。 | [src/vulkan/mod.rs](../../../inferences/RustingStrata/src/vulkan/mod.rs)、[src/vulkan/shaders/gemmi.glsl](../../../inferences/RustingStrata/src/vulkan/shaders/gemmi.glsl)、[README.md](../../../inferences/RustingStrata/README.md) |
| Shader资源预检 | preflight通过SPIR-V及specialization算workgroup shared memory，上传权重前验证模型所需核；部分pipeline预建减少首步停顿。 | 硬件报告与pipeline专用资源都要核对；预检不能代替真卡输出正确性。 | [src/vulkan/preflight.rs](../../../inferences/RustingStrata/src/vulkan/preflight.rs)、[src/vulkan/spv_smem.rs](../../../inferences/RustingStrata/src/vulkan/spv_smem.rs) |
| 完整模型以mmap backing | Weights/Tensor按GGUF布局引用量化字节；CPU miss从文件映射和OS page cache访问，不另构建全模型FP32副本。 | mmap不等于模型已在RAM，也不是自有SSD专家缓存；page cache竞争直接影响表现。 | [src/weights.rs](../../../inferences/RustingStrata/src/weights.rs)、[src/gguf.rs](../../../inferences/RustingStrata/src/gguf.rs) |
| 按RAM容量预读及网络盘保护 | fits_ram根据已映射字节+4GiB与MemTotal判断；后台分128KiB的MADV_WILLNEED读入，检测Linux网络挂载默认不整文件预读。 | 使用MemTotal而非MemAvailable是源码已知限制；预读完成标志表示建议已发出，不表示磁盘读取全部完成。Windows/NAS检测覆盖不能推定与Linux一致。 | [src/weights.rs](../../../inferences/RustingStrata/src/weights.rs)、[src/memory.rs](../../../inferences/RustingStrata/src/memory.rs) |
| 多种预取目标不能混称 | Prefetch把mmap范围预读到OS cache；cold_ranges_for_next选择下层冷专家；另有GPU L2 moe_prefetch。 | 前者不是H2D，后者不是读SSD；L2预取需STRATA_MOE_PREFETCH显式开启，部分小L2设备禁用以免驱逐当前工作集。 | [src/weights.rs](../../../inferences/RustingStrata/src/weights.rs)、[src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs)、[src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs) |
| 同大小专家分池、压缩布局驻留 | slot包含gate/up/down且16B对齐，按byte size分池并维护每层expert→slot表；池的全容量分组受2GiB界限约束。 | 预算按比例分配池，热点可跨同size层竞争；不是所有层有固定相同命中率或配额。 | [src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs) |
| 模型状态优先于专家缓存 | 先分配dense/state/KV/prefill scratch，再从剩余显存扣margin给专家；分配失败减少10%重试。 | margin/ctx/parallel/chunk共同决定命中率；不能只报“专家缓存用了多少”。显式EXPERT_MB可覆盖预算。 | [src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs)、[src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs) |
| 短长两种热点统计 | counts每adapt保留0.95，long保留0.999；prompt路由独立累计，长期profile主要跟随decode。 | 避免长prompt淹没短reply，也避免只存末条reply导致下轮placement摆动；profile key不是完整权重hash。 | [src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs) |
| 带门槛的热点换入 | 默认通用load每8token、每池最多64 swaps；miss热度超过resident的1.5倍+2才换，减少抖动。 | serve的set_decode_only改为每16token/16 swaps并重置prompt影响；bench与serve默认不是同一组参数。 | [src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs)、[src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs) |
| Prefill后缓存策略及反例 | settle将prompt统计并入短期表；PREFILL_PROTECT可限制权重及换入数量，strict不换。 | PREFILL_PROTECT默认关闭；源码记录保护旧decode缓存反而降低后续命中，不能无条件认为保留旧热点更好。 | [src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs) |
| 全专家驻留快路径 | fully_resident且后端允许时跳过mailbox请求、CPU计算和不必要统计；未记录路由时不覆盖旧profile。 | worker对象可能仍被分配；快路径省执行成本，不等于所有相关对象均消失。 | [src/expert_cache.rs](../../../inferences/RustingStrata/src/expert_cache.rs)、[src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs) |
| CPU/GPU mailbox专家协作 | GPU发布路由、slot、activation与REQ，CPU worker只算miss，GPU combine仅在miss时等DONE；CUDA无miss发SKIP。 | 需要CPU/GPU内存可见性、seq防旧请求、独立cache line及正确fence；残差依赖仍存在，不能任意同时跑有依赖层。 | [src/cpu_worker.rs](../../../inferences/RustingStrata/src/cpu_worker.rs)、[src/cuda/kernels.cu](../../../inferences/RustingStrata/src/cuda/kernels.cu) |
| CPU压缩点积和共享准备 | CPU activation按Q8_K量化，格式专用整数点积；gate/up专家批次共用准备并减少Rayon唤醒，部分格式有四activation共读一行的核。 | 缺整数核走浮点解码；generic expert_rows并非所有重复expert都显式去重，不能从批处理推断所有路径权重只读一次。 | [src/cpu_moe.rs](../../../inferences/RustingStrata/src/cpu_moe.rs) |
| SIMD与线程数的数值约束 | AVX2代码受编译target_feature控制；AVX512 VNNI/VBMI另需开关，CPU池倾向物理核以减SMT带宽竞争。 | 项目.cargo默认target-cpu=native；AVX512求和顺序可能破坏draft/plain近tie一致性，默认不自动视为无损快路径。 | [src/cpu_moe.rs](../../../inferences/RustingStrata/src/cpu_moe.rs)、[.cargo/config.toml](../../../inferences/RustingStrata/.cargo/config.toml)、[src/main.rs](../../../inferences/RustingStrata/src/main.rs) |
| 有界GPU等待及断点恢复 | Vulkan combine共享verdict与SPINS预算，host超时可让出，conditional rendering阻止后续读半成品；GpuModel.finish等CPU后从该combine继续。 | 没有conditional rendering或coherent mailbox时可能报错；同层再次看不到答案明确拒绝。目标是避开driver watchdog，不是丢弃CPU结果继续生成。 | [src/vulkan/shaders/mb_wait.glsl](../../../inferences/RustingStrata/src/vulkan/shaders/mb_wait.glsl)、[src/vulkan/mod.rs](../../../inferences/RustingStrata/src/vulkan/mod.rs)、[src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs) |
| 按完成事件保护host staging ring | CUDA大H2D先复制到pinned ring，重叠区域复用前等事件；HIP当前也实现同类ring。 | ring让caller源可复用但DMA仍在途；小pageable拷贝路径依runtime staging语义。HIP文档仍写ring未移植，源码为准。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/hip/mod.rs](../../../inferences/RustingStrata/src/hip/mod.rs)、[docs/HIP.md](../../../inferences/RustingStrata/docs/HIP.md) |
| 受依赖约束的并发API | batch仅用于无相互依赖ops，CUDA轮流放主/侧stream并join；beside让本组拷贝与计算及下组重叠，最终beside_join收齐。 | Vulkan用可用transfer queue和timeline semaphore；HIP未覆盖beside则使用trait串行fallback，不能一律宣称双stream。 | [src/backend.rs](../../../inferences/RustingStrata/src/backend.rs)、[src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/vulkan/mod.rs](../../../inferences/RustingStrata/src/vulkan/mod.rs)、[src/hip/mod.rs](../../../inferences/RustingStrata/src/hip/mod.rs) |
| Grouped prefill与分组搬运 | prompt按chunk逐层，(token,expert)按专家整理成GEMM tile；miss每组16专家，通过三组staging轮转，与前组计算重叠。 | prefill多token时与decode CPU miss分工不同；staging、tile索引和activation需计入预算。 | [src/gpu_model/prefill.rs](../../../inferences/RustingStrata/src/gpu_model/prefill.rs) |
| 生命周期scratch别名 | Scratch的mixer/FFN共享slab区间，HC/PLE临时区在不重叠阶段复用，降低峰值分配。 | 静态layout依赖执行顺序；任意增加并发可能使别名失效，不能只照搬offset。 | [src/gpu_model/prefill.rs](../../../inferences/RustingStrata/src/gpu_model/prefill.rs) |
| 同输入量化复用和小算子融合 | SAME_X按输入地址/stride/rows/cols复用Q8；prefill显式prequant gate供beta/alpha共用；融合norm/RoPE、SwiGLU及部分expert combine。 | 地址相同不足以保证内容相同，调用方契约是输入未变；并发准备必须先完成再让多消费者读。 | [src/backend.rs](../../../inferences/RustingStrata/src/backend.rs)、[src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/gpu_model/prefill.rs](../../../inferences/RustingStrata/src/gpu_model/prefill.rs)、[src/cuda/kernels.cu](../../../inferences/RustingStrata/src/cuda/kernels.cu) |
| 量化内算和tile流水 | sm80+适用格式量化activation进INT8 MMA，权重tile解码；FP16 GEMM另路，cp.async双缓冲隐藏设备内存读。Vulkancoop/dot路径对齐归约。 | INT8激活引入与FP32参考的差异，--llama-numerics可选CPU q8_K契约；FP4权重格式不意味着每卡都有FP4 MMA。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/cuda/kernels.cu](../../../inferences/RustingStrata/src/cuda/kernels.cu)、[src/vulkan/shaders/gemmi.glsl](../../../inferences/RustingStrata/src/vulkan/shaders/gemmi.glsl) |
| Attention按能力与形状分块 | CUDA有64/128/256 head dimension prefill核与拆分decode；Vulkan选coop、tiled或per-query；按shared memory限制tile。 | mask/累加顺序/精度须核对；shared memory几字节越界也可导致pipeline失败，长ctx另需整数范围检查。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/vulkan/preflight.rs](../../../inferences/RustingStrata/src/vulkan/preflight.rs)、[src/vulkan/shaders/attn_prefill.glsl](../../../inferences/RustingStrata/src/vulkan/shaders/attn_prefill.glsl) |
| KV低位选择 | CUDA/Vulkan有f16/q8/q4/q2的cache布局和对应attention；不同codec减少长context容量与读带宽。 | KV quant改变数值；HIP当前kv_words仅f16/q8，后端支持不能由共享接口推定。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/vulkan/mod.rs](../../../inferences/RustingStrata/src/vulkan/mod.rs)、[src/hip/mod.rs](../../../inferences/RustingStrata/src/hip/mod.rs) |
| 记录图并限制CUDA/HIP数量 | CUDA graph按模型调用key复用，动态参数从device step_buf读；CUDA/HIP已有32个图后新形状eager执行；Vulkan复用命令缓冲。 | 不是统一LRU/floor策略；Vulkan的所读graph实现未见同样32上限。slot组合增加会影响容量与命中。 | [src/cuda/mod.rs](../../../inferences/RustingStrata/src/cuda/mod.rs)、[src/hip/mod.rs](../../../inferences/RustingStrata/src/hip/mod.rs)、[src/vulkan/mod.rs](../../../inferences/RustingStrata/src/vulkan/mod.rs)、[src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs) |
| 多请求合批、状态准入与停放 | step_rows将多个slot一forward，通用dense还可step_batch；当前活跃state占VRAM，saved/parked尽量放host，按RAM cap淘汰。 | live slots不够时其余请求排队；Vulkan部分saved state仍在VRAM，源码标注预算遗漏风险；不能把全部slot状态称RAM常驻。 | [src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs)、[src/main.rs](../../../inferences/RustingStrata/src/main.rs) |
| 前缀/递归状态持久化 | Parked包含tokens、owner、PLE位置、递归和KV；最多最近4个状态可原子写文件，按fingerprint拒绝不兼容加载。 | 所读fingerprint偏几何/KV尺寸，不含完整权重或LoRA hash；用于Comfy缓存应补齐真实依赖，不能把该key当完整身份保证。 | [src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs) |
| 投机多proposal与回滚 | 支持MTP/EAGLE3/DFlash/外部draft和suffix/ngram lookup，target批量verify，恢复KV/conv/SSM等未接受状态；partial MoE可缩短或暂停MTP。 | 主要是LLM优化；各模型/backend/采样能力不同。CPU/GPU placement和BATCHED归约可能改变答案，不承诺跨配置等同。 | [src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs)、[src/ngram.rs](../../../inferences/RustingStrata/src/ngram.rs)、[src/raon/dflash.rs](../../../inferences/RustingStrata/src/raon/dflash.rs)、[src/inkling2/dspark.rs](../../../inferences/RustingStrata/src/inkling2/dspark.rs) |
| 按资源/参数搜索的预算化校准 | coarse低差异采样→多起点坐标下降→cheap polish，默认24样本，含失败trace、deadline及2% baseline胜出门槛。 | 几何均值折中prefill/decode不是每指标无回退保证；聚合跳过无效measurement，需另验完整工作负载和画质。 | [src/calibrate.rs](../../../inferences/RustingStrata/src/calibrate.rs)、[src/main.rs](../../../inferences/RustingStrata/src/main.rs)、[docs/CALIBRATION.md](../../../inferences/RustingStrata/docs/CALIBRATION.md) |
| 固定用户资源约束和可复用校准 | 当前calibrate不搜索ctx/KV，不搜索多卡split；按model/backend/device存配置，并记录语料/深度，bench子进程统一测时。 | main的Cmd::Calibrate忽略split并drop旧split；README/THANKS部分表述较旧。校准key/已验证元数据不足以替代权重和driver完整指纹。 | [src/main.rs](../../../inferences/RustingStrata/src/main.rs)、[docs/CALIBRATION.md](../../../inferences/RustingStrata/docs/CALIBRATION.md) |
| 多卡按层切分与实际传输 | GpuModel链分配连续层，容量初始按free VRAM，可显式份额；pass/rows_pass同步并把activation读host再送下一卡。 | 不是自动P2P Tensor Parallel；CPU往返和设备降频可能抵消收益；HIP当前拒绝多设备。 | [src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs)、[src/backend.rs](../../../inferences/RustingStrata/src/backend.rs) |
| 专用模型的渐进GPU迁移 | DeepSeek Mat层可将非expert矩阵驻留GPU，CPU控制forward按调用上传activation/下载结果；GLM等复用相关cache/批处理，Inkling另有device forward。 | 不同family实现成熟度不同；DeepSeek当前prefill无通用chunk搜索，不能用通用引擎GPU驻留结论覆盖全部模型。 | [src/deepseek/mat.rs](../../../inferences/RustingStrata/src/deepseek/mat.rs)、[src/deepseek/forward.rs](../../../inferences/RustingStrata/src/deepseek/forward.rs)、[src/glm/forward.rs](../../../inferences/RustingStrata/src/glm/forward.rs)、[src/inkling2/forward/device.rs](../../../inferences/RustingStrata/src/inkling2/forward/device.rs) |
| LoRA兼容与快路径退让 | Weights可烘焙部分adapter，GPU down-expert delta路径能读回activation在host补偿；存在LoRA时部分graph路径不捕获。 | 兼容成本可能包含D2H/H2D与同步；不能把“支持LoRA”解释为融合核或图仍全速。 | [src/weights.rs](../../../inferences/RustingStrata/src/weights.rs)、[src/gpu_model.rs](../../../inferences/RustingStrata/src/gpu_model.rs)、[docs/LORA.md](../../../inferences/RustingStrata/docs/LORA.md) |
| 空闲释放与失败后服务状态 | auto-unload空闲释放模型，下次chat重载；device-lost错误有类型，HTTP状态区分已卸载可重载与不可服务。 | 重载延迟属于冷/温启动；GPU丢失不能继续接受并返回假正常的推理。 | [src/main.rs](../../../inferences/RustingStrata/src/main.rs)、[src/serve.rs](../../../inferences/RustingStrata/src/serve.rs) |

## 必须保留的限制与负面经验

- **统计目标随入口变化。** 通用load的8token/64swap与serve的16token/16swap不同，prefill后还可重置短期counts。bench最优不自动等于生产默认最优。
- **“固定专家”不等于固定全部实验条件。** 本地ExpertCache构造在设置`STRATA_FIXED_EXPERTS`时把profile设None，不读该profile，也不自适应，初始填充回到层序；其他family有自己的实现。预算/模型/内核/页缓存仍会影响结果，不能只加这个开关便宣称公平对照。
- **异步等待必须能退出。** 对冷盘CPU miss一直在GPU里spin，可能先撞driver watchdog。Vulkan的让出/条件跳过/等待CPU/从combine恢复是整体协议，不能只拷贝timeout阈值。
- **状态key尚非完整依赖指纹。** 专家profile以arch/层数/expert数/tensor数命名，持久prefix fingerprint主要是几何和KV结构；当前校准也不完整绑定driver/权重。用于跨模型、LoRA和ComfyUI条件缓存时，须扩展身份或限制复用范围。
- **主存也要预算。** preload用MemTotal而非MemAvailable；host saved/parked、mmap page cache、worker临时tensor会竞争。源码还标出Vulkan saved state预算遗漏及部分Weights合成小buffer泄漏的已知限制，不应把低VRAM等同于低整机内存。
- **文档默认可能落后。** HIP.md仍写pinned ring尚未移植，当前HIP htod已有ring；但HIP没有独立beside override，不能据此进一步推导它也有CUDA式side stream。校准旧说明中的多卡split搜索同样需要以当前Cmd::Calibrate和drop_split为准。

## 数值与性能结论如何使用

CPU Q8_K、GPU量化activation、不同attention tile和backend归约可能不同；`BATCHED`明确允许多请求走另一求和路径，AVX512默认opt-in也与数值有关。投机算法设计上验证target分布，不证明它与另一placement、另一batch或原BF16模型相同。greedy、sampling、prefill和跨请求复用分别验收。

README披露dense多请求对比使用Q4_K_M，而vLLM用BF16，不能据此作同质量速度排名；大MoE的文件/page cache接近RAM极限，项目自报运行间可有约30%波动。本轮不引用这些数字作为目标硬件承诺。

## 对ComfyUI的借鉴

| 方向 | 可以学习的具体内容 | 迁移边界 |
| --- | --- | --- |
| 执行器内部并发 | Backend把独立算子batch与copy/compute beside分开，依赖明确后再join | 比把工作流全部节点并发更细；主算子已占满GPU时不一定获益 |
| 显存规划 | 先算状态/scratch/预取槽，再给常驻权重；静态slab按阶段别名 | DiT宜按确定层序预取，不照搬MoE热点和CPU miss策略 |
| 启动时间 | 编译到机器码并按源码/架构/编译器版本缓存 | 首次compile、冷load、热run分开；Vulkan预编译SPIR-V与runtime pipeline又是另一层缓存 |
| 硬件覆盖 | shader specialization实际shared-memory预检，小L2预取开关、旧卡同步拷贝fallback | 看实际能力，不由支持某API推定吞吐和全部算子兼容 |
| 稳定性 | GPU等待有界、在途内存可见性、错误后恢复或停止服务 | 主机/driver timeout与cancel都要纳入，不能带着半成品继续 |
| 自动调优 | 有预算的coarse→refinement，基准取生产默认，失败也留trace，ctx/KV不暗改 | 综合得分外再加每工作负载、p95和画质约束；不把2%门槛当统计显著性证明 |
| 验证 | 冻结placement/codec/shape/采样，检查batch宽度和缓存依赖 | 精确记录实际生效参数、页缓存和首轮状态，不能只固定seed |

建议先在ComfyUI已存在的预取/图路径上观测scratch、拷贝、首轮compile和shape特化，不先引入一套新后端抽象。稀疏专家CPU offload、KV codec和token draft主要对应工作流内的文本模型；dense扩散若把层放CPU，多次forward的算力与搬运成本需独立评估。

## 代码复用与后续跟踪

项目自有代码采用RustingStrata License 1.0，非Strata的MIT；声明允许使用/修改/集成和部分商业服务，同时限制实质上销售该引擎或其替代品，并要求署名及相应分发声明。这里只记录本地许可证内容；直接复制到产品前按具体文件和分发方式核对，第三方dot/compat代码仍保留自身声明。

本轮读了目录/配置、通用load/prefill/step/graph、专家cache/worker/CPU dot、CUDA/HIP/Vulkan资源和队列、校准与主CLI连接、代表性专用forward、ngram、状态缓存和server生命周期。未逐行读完所有模型、HTTP/UI/MCP、量化表、全部CUDA/GLSL核或SPIR-V二进制；未运行仓库单测或parity。后续按目标模型补查具体forward/format/backend组合，尤其持久cache身份、内存约束、Vulkan恢复和HIP并发覆盖。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI迁移与实验清单](../COMFYUI.md)
- [统一版本与采集记录](../SNAPSHOTS.md)
