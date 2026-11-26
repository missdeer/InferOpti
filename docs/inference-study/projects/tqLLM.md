# tqLLM：量化格式、成本分派与分层执行

研究日期：2026-11-26。本轮为静态源码研究，未构建、未运行模型或重测 benchmark。

- 完整 commit：`7df840632cd3f108de459e0944b9b6d115532689`
- 提交时间：`2026-11-25T22:53:14+00:00`
- 提交主题：tqLLM 1.8.3: first drafted reply at full speed (#106), stable dense kernels (#105), faster Mac experts (#109)
- 本地分支：`main`；源码工作区 tracked clean、untracked 0。
- 源仓库：https://github.com/turbo-tan/tqllm.git
- 源码状态与采集时间见[统一版本快照](../SNAPSHOTS.md)。

## 结论

tqLLM 最值得借鉴的是：**量化格式与 kernel/布局一体化；按格式、矩阵形状和设备成本选择执行路径；显存适配与图/状态/缓存共同规划；自适应策略覆盖首次和热请求。** 当前提交是1.8.3，README主要仍以1.8.0介绍，版本判断以HEAD与CHANGELOG/源码为准。

它包含通用C++接口、CUDA执行器、基于MLX和自定义Metal的Apple执行器，以及多个专用模型。Android目录不能据此算作已交付后端。没有在本轮读到已接通ComfyUI的DiT/VAE生成路径；对ComfyUI的价值主要是机制借鉴，不能直接作为图像生成替代后端。

## 方法与策略

下面“源码可见”仅说明存在相应实现；入口、默认值、模型适配和真机收益需要分别确认。表中代码链接均对应以上commit；固定版本远端链接另存于[统一源码证据](../evidence.csv)。

| 方法 | 实现与优化目的 | 条件、默认值与边界 | 源码证据 |
| --- | --- | --- | --- |
| 格式、布局、激活基底共同构成计算契约 | WeightFormat 声明 Plain/Tq3Rot32/Tq4Rot128、matvec/GEMM、row slicing 与 expert 能力；相同输入的旋转/准备可被多个矩阵共享。 | 能够解析 GGUF 不代表 kernel 支持所有形状；Flash-Next 仍有绕过统一注册表的历史路径。 | [src/formats/weight/weight_format.hpp](../../../inferences/tqLLM/src/formats/weight/weight_format.hpp)、[src/formats/weight/registry.cpp](../../../inferences/tqLLM/src/formats/weight/registry.cpp)、[AGENTS.md](../../../inferences/tqLLM/AGENTS.md) |
| 加载时重排，减少每步解码开销 | TQ3_4S 装载后 repack 到 PRMT 友好布局并 tune；TQ4/TQT 把 code 与 scale 分开，便于行切片。 | 重排是启动成本；不要在每步 forward 再做。TQ3_4S 每 32 权重 16 字节，整体为 4 bpw，名称中的 3 不等于总存储 3 bpw。 | [src/formats/weight/tq3_4s.cpp](../../../inferences/tqLLM/src/formats/weight/tq3_4s.cpp)、[src/formats/weight/quant_tq3.inc](../../../inferences/tqLLM/src/formats/weight/quant_tq3.inc)、[src/formats/weight/tq4.cpp](../../../inferences/tqLLM/src/formats/weight/tq4.cpp) |
| 旋转低位 codebook 与 trellis | TQ4 每 128 权重为 64 字节 code + 2 字节 scale（4.125 bpw）；TQT 同大小改用 tail-biting trellis，运行时 table-free hash 生成权重对。 | TQ、TQT、TQV 不是可互换的通用 INT4；与 Sync-Strata 的 TQ 也不能只凭名字判断兼容。 | [src/formats/weight/tq4.cpp](../../../inferences/tqLLM/src/formats/weight/tq4.cpp)、[src/formats/weight/tqt.cpp](../../../inferences/tqLLM/src/formats/weight/tqt.cpp)、[src/formats/weight/tqt_decode.cuh](../../../inferences/tqLLM/src/formats/weight/tqt_decode.cuh) |
| 用真实激活校准量化 | 捕获输入，分块计算 H=XᵀX；按模型基底旋转 H/W，用 GPTQ、role damp、block act-order、scale refit，TQV 可另开 ICM。 | H 本身是 K² 内存，离线转换不等于免费；必须用目标任务代表性数据和 held-out 质量。 | [tools/convert/qwen35/tqv12_quant.py](../../../inferences/tqLLM/tools/convert/qwen35/tqv12_quant.py) |
| 把编码复杂度留在离线端 | TQT encoder 用 CUDA Viterbi 搜索 trellis；写回后独立解码抽样行，检查与量化后的权重完全对应。 | Viterbi 分支 metric 用 FP16，注释注明 near-tie 时与高精度搜索可不同；回读一致不等于与原始权重无损。 | [tools/convert/qwen35/tqt_viterbi.cu](../../../inferences/tqLLM/tools/convert/qwen35/tqt_viterbi.cu)、[tools/convert/qwen35/tqv12_quant.py](../../../inferences/tqLLM/tools/convert/qwen35/tqv12_quant.py) |
| 压缩权重直接算，不先展开整矩阵 | TQ3_4S、TQ4、TQT 均有 native GEMM/matvec；TQ4 代码可见 f16 MMA，权重在 tile 内解码。 | 低位存储不等于低位 MMA：这个路径不能写成原生 INT4 Tensor Core。 | [src/formats/weight/tq4.cu](../../../inferences/tqLLM/src/formats/weight/tq4.cu)、[src/formats/weight/tq4.cpp](../../../inferences/tqLLM/src/formats/weight/tq4.cpp)、[src/formats/weight/tqt.cpp](../../../inferences/tqLLM/src/formats/weight/tqt.cpp) |
| Prefill 路径按成本交点选择 | prefill_crossover 比较 batched matvec、native GEMM 与 dequant+cuBLAS 的固定和逐列成本；prefill_path 决定实际分派。 | 有/无 FP16 scratch、verify 模式、格式能力都参与；小块/慢带宽卡可能因反复展开而慢。需追到 load 里的实测校准。 | [src/models/qwen/qwen35/prefill_path.hpp](../../../inferences/tqLLM/src/models/qwen/qwen35/prefill_path.hpp)、[src/models/qwen/qwen35/qwen35_load.cpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_load.cpp) |
| 可重复的 kernel 分派规则 | matvec_rules 按格式、compute capability、K、rows、columns 选 Legacy/Split16、prefetch 和 tile 形状，规则带来源。 | TQ3 两族舍入不同，选择固定到设备/形状，不做每次启动噪声驱动的随机切换；同名格式也可能多种数值路径。 | [src/formats/weight/matvec_rules.hpp](../../../inferences/tqLLM/src/formats/weight/matvec_rules.hpp) |
| 可选 INT8/FP8 prefill | 按行权重/按 token 激活 scale，INT8 路径有 SmoothQuant 列 scale；可直接从压缩权重产 W8，避免 FP16 往返。 | 自动 INT8/FP8 条件包含 !exact；显式 on 可以覆盖 exact 并警告。属于数值/质量实验。 | [src/core/int8_gemm.cu](../../../inferences/tqLLM/src/core/int8_gemm.cu)、[src/models/qwen/qwen35/qwen35_load.cpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_load.cpp)、[src/core/options.cpp](../../../inferences/tqLLM/src/core/options.cpp) |
| Scratch 别名和准备结果常驻 | INT8 权重区能放入原 FP16 scratch 空闲部分，cuBLAS/cuBLASLt 共用 workspace；足够内存时可留 FP8 权重副本省重复转换。 | 保留副本的内存门槛很高；不是所有小卡都可走，也不能忽略副本大小。 | [src/models/qwen/qwen35/qwen35_load.cpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_load.cpp) |
| 混合格式矩阵分支并发 | Composite 各 part 读同一输入，matvec_ld 直接写输出不同 row range；side streams fork/join，不再拼接临时输出。 | 仅互不覆盖的分块；静态 pool 假定主 stream 一次驱动一个 composite，不可推成任意并发模型线程安全。 | [src/formats/weight/composite.cpp](../../../inferences/tqLLM/src/formats/weight/composite.cpp) |
| 按剩余容量调整 prefill chunk | context、总显存、KV 占用后 room 决定 chunk；大机器可用更大块减少重复访问，小卡控制激活峰值。 | 阈值带具体硬件历史；更大 chunk 改变 kernel/归约路径，缓存复现还依赖 chunk 边界。 | [src/models/qwen/qwen35/prefill_chunk.hpp](../../../inferences/tqLLM/src/models/qwen/qwen35/prefill_chunk.hpp) |
| 自动 fit：容量不足时分步退让 | load_fitted 可把 embedding/head、KV 移到 host，再调整 prefill buffer/权重分块、KV codec 和 context；用一次容量测量推算 placement，减少重复 reload。 | fit_lossy 默认 true；-fitl off 才拒绝需要变更 codec/context/prefill 配置的装载。用户显式配置的处理逐项看代码，不能把所有步骤都叫无损 offload。 | [src/backends/cuda/load_fitted.cpp](../../../inferences/tqLLM/src/backends/cuda/load_fitted.cpp)、[src/engine/fit_context.hpp](../../../inferences/tqLLM/src/engine/fit_context.hpp)、[include/tqllm/options.hpp](../../../inferences/tqLLM/include/tqllm/options.hpp) |
| WDDM/UMA 的内存口径 | WSL/WDDM 使用加载前占用+引擎计数+slack；UMA 另考虑可回收页缓存，保留 graph/slot 余量。 | 快照 free 不能代表一直可用；某些 fallback 依据平台/设备特征，不保证其他机器同样可靠。 | [src/backends/cuda/load_fitted.cpp](../../../inferences/tqLLM/src/backends/cuda/load_fitted.cpp)、[src/engine/fit_context.hpp](../../../inferences/tqLLM/src/engine/fit_context.hpp) |
| KV 编码与位置分层分开 | KvFormat 定义 codec/attention，KvPool/KvStore 定义 flat/paged/tiered placement；CUDA VMM 在稳定地址映射 host/device granule。 | 默认 flat，paged/tiered 是可选 policy；AGENTS 明说 Flash-Next 尚有自己 K/V buffers，统一化没有覆盖所有模型。 | [src/kv/kv_store.hpp](../../../inferences/tqLLM/src/kv/kv_store.hpp)、[src/kv/kv_store_cuda.cpp](../../../inferences/tqLLM/src/kv/kv_store_cuda.cpp)、[AGENTS.md](../../../inferences/tqLLM/AGENTS.md) |
| KV host tail 每层 DMA 回 staging | staged 模式把旧位置搬 host，用两组按 attention-layer 奇偶共享的 staging 物理页映射旧地址；该层 attention 前回填。 | 单槽/格式/几何有条件；remap 前等在途拷贝，失败回滚映射，无法恢复则终止，避免留下可读空洞。 | [src/kv/kv_store.hpp](../../../inferences/tqLLM/src/kv/kv_store.hpp)、[src/kv/kv_store.cpp](../../../inferences/tqLLM/src/kv/kv_store.cpp)、[src/kv/kv_store_cuda.cpp](../../../inferences/tqLLM/src/kv/kv_store_cuda.cpp) |
| 低位 KV、不对称 K/V 与近似 compact | tq3 K/V、tq3k-tq2v 有配套写入/attention；kv-compact 可把旧 8K 块拟合成更小 stand-in。 | tq3 路径要求 hd=256；换 codec 与 compact 都改变数值。compact 默认0、opt-in，不能与保持原编码的 host 搬运混为一谈。 | [src/formats/kv/kv_tq3.cpp](../../../inferences/tqLLM/src/formats/kv/kv_tq3.cpp)、[src/formats/kv/kv_tq3k_tq2v.cpp](../../../inferences/tqLLM/src/formats/kv/kv_tq3k_tq2v.cpp)、[src/models/qwen/qwen35/qwen35_forward.cpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_forward.cpp)、[include/tqllm/options.hpp](../../../inferences/tqLLM/include/tqllm/options.hpp) |
| QSA 稀疏 attention | 借助文件携带的训练 indexer，挑选历史 key blocks，近端/未提交范围按固定规则密集算以保持 tree 与单 token 的行一致。 | 相对原 dense attention 是近似；CUDA qsa-from 默认策略与 Metal 文件元数据不同，需额外 indexer 权重，不是任意模型的免费稀疏化。 | [src/backends/metal/attention_qsa.hpp](../../../inferences/tqLLM/src/backends/metal/attention_qsa.hpp)、[include/tqllm/options.hpp](../../../inferences/tqLLM/include/tqllm/options.hpp) |
| 专家三级缓存与准备布局复用 | MixedExpertCache 在 GPU banks 存格式布局；HostTier 存已 repack/tile 的 pinned blocks；miss 再从 GGUF/冷 store 读。 | RAM 宽裕时保留副本可省 D2H，压力下优先回收 GPU 已持有的 host block；不是绝对永远 exclusive。 | [src/experts/mixed_expert_cache.cpp](../../../inferences/tqLLM/src/experts/mixed_expert_cache.cpp)、[src/experts/host_expert_tier.hpp](../../../inferences/tqLLM/src/experts/host_expert_tier.hpp)、[src/models/qwen/qwen4exp/fn_host_tier.hpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/fn_host_tier.hpp) |
| 按层配额与弹性借还 | 每层 target/floor 分配 slots，给尚未建 bank 的层公平留容量；长 prompt 借 banks，RAM tier 弹性块可暂退给 prompt host buffers。 | 避免前几层占满后迫使后层相互驱逐；refill/重建额外空间和峰值也要计入。 | [src/experts/mixed_expert_cache.cpp](../../../inferences/tqLLM/src/experts/mixed_expert_cache.cpp)、[src/models/qwen/qwen4exp/memory_plan.hpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/memory_plan.hpp)、[src/models/qwen/qwen4exp/fn_engine_load.cpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/fn_engine_load.cpp) |
| 下一层专家预测、有限预取深度 | 从当前中间输入估计下一层路由，一层提前搬猜中 miss；park 与 demand 分离，预测排队有限额，真实 demand 优先。 | 预测只改 placement，不替代真实 router；多请求更保守，错误猜测仍会吃 PCIe/slot。kGuessLinkDepth=5、park=16 是当前校准值。 | [src/experts/expert_prediction.hpp](../../../inferences/tqLLM/src/experts/expert_prediction.hpp)、[src/experts/expert_fetch_queue.hpp](../../../inferences/tqLLM/src/experts/expert_fetch_queue.hpp)、[src/models/qwen/qwen4exp/fn_layer_forward.cpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/fn_layer_forward.cpp) |
| Host 缓存 epoch/pin/fence 协议 | 读取和写入都关联 copy epoch；同一 open epoch 不能复用 block；跨 epoch 完成事件/fence 后才回收。 | release 后仍可暂保映射防止不必要的回拷；“已释放逻辑引用”不等于 DMA 已结束。 | [src/experts/host_expert_tier.hpp](../../../inferences/tqLLM/src/experts/host_expert_tier.hpp) |
| 专家 dedup、grouped GEMM 与 CPU miss lane | 同一步按 unique expert 聚合，gate/up 可成对，避免为每 assignment 重读权重；部分路径用 CPU pool 算 miss。 | CPU miss 被 exact_conflicts 列为近似；verify 分 waves 的归约不同，distinct expert 是否装得下会影响路径。 | [src/models/qwen/qwen4exp/fn_layer_forward.cpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/fn_layer_forward.cpp)、[src/models/qwen/qwen4exp/decode_rules.hpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/decode_rules.hpp)、[src/core/options.cpp](../../../inferences/tqLLM/src/core/options.cpp) |
| Cold store 与精度/驻留独立 | hot map 是精度/放置策略而非 router mask；cold record 绑定 encoder revision、源文件与精度图，并校验/缓存 mmap。 | 保留真实专家ID，不等于冷专家可省略；checksum 仅验证格式身份和数据一致性。 | [src/experts/hot_cold_policy.hpp](../../../inferences/tqLLM/src/experts/hot_cold_policy.hpp)、[src/experts/cold_expert_store.hpp](../../../inferences/tqLLM/src/experts/cold_expert_store.hpp) |
| PCIe 与一致性 host memory 区分 | zero_copy_guard 对需要 device 指针的 kernel，在普通 PCIe 卡拒绝 bulk host 指针；coherent host-page-table 设备有另外路径。 | 小型 embedding/head gather 有显式例外。省 memcpy 不能推断快，也不能推断所有零拷贝访问都被禁止。 | [src/core/zero_copy_guard.hpp](../../../inferences/tqLLM/src/core/zero_copy_guard.hpp) |
| Inkling 驻留计划持续自适应 | ReplanClock 同时统计 plain token 和 drafted verify block，初始每32步重规划；移动少则指数退避，最多1024步，新 prompt 重置。 | 只在 partial residency 生效；每32步都搬会有维护成本，不能只优化热请求。 | [src/models/inkling2/loop/inkling2_residency.hpp](../../../inferences/tqLLM/src/models/inkling2/loop/inkling2_residency.hpp)、[src/models/inkling2/loop/inkling2_model.cpp](../../../inferences/tqLLM/src/models/inkling2/loop/inkling2_model.cpp) |
| 图缓存限额、热工作集与owner身份 | GraphLru 容量/低内存双约束，保留 floor 防反复捕获；destroy 前 quiesce；OOM 可清理后重试。Flash-Next 用新 owner ID 防地址重用误命中。 | 默认 cache=256、floor=24 是当前实现；缓存图和数据 arena 都占显存。不能把 pointer 地址当完整身份。 | [src/core/graph_lru.hpp](../../../inferences/tqLLM/src/core/graph_lru.hpp)、[src/models/qwen/qwen4exp/decode_rules.hpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/decode_rules.hpp)、[include/tqllm/options.hpp](../../../inferences/tqLLM/include/tqllm/options.hpp) |
| 前缀状态按 chunk/消息边界复用 | 保存 last-message/system/turn anchor 的 K/V 和 recurrent snapshot；匹配最长合法前缀，避免 chat template 删除旧 reasoning 后每轮重读。 | exact 下不随意复用 live decode 末尾，因为 cold prefill 归约可能不同；缓存本身可能改变首次 chunk 划分。 | [src/models/qwen/qwen35/prefill_chunk.hpp](../../../inferences/tqLLM/src/models/qwen/qwen35/prefill_chunk.hpp)、[src/models/qwen/qwen35/qwen35_forward.cpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_forward.cpp) |
| PromptPark 把状态从 slot 解耦 | slot 换会话时将 kept state 停放 host；优先空槽、dead、未被排队请求保护的最旧状态；拒绝覆盖受保护状态。 | 这里是纯索引策略，拷贝由 engine 做，host parked state 仍占 RAM。 | [src/kv/prompt_park.hpp](../../../inferences/tqLLM/src/kv/prompt_park.hpp) |
| 投机只追净收益，失败后退避重试 | Inkling policy 拟合 time=a+bL，用逐深度接受率或校准后的 draft confidence 选窗口；若比 plain 慢，退回普通步后再探测。 | DFlash2 可能仍先生成整块再决定验证宽度；不能把少验证行当作相同幅度减少全部 draft 成本。 | [src/models/inkling2/loop/inkling2_spec_policy.hpp](../../../inferences/tqLLM/src/models/inkling2/loop/inkling2_spec_policy.hpp) |
| Mac 宽度策略不让 JIT 污染成本 | WidthPolicy 按7/15 draft宽度的预期 tokens/time 选择，主动探索未观察的尾部；排除每请求首步建图和进程每宽度首次 JIT。 | context>=8000 时当前策略固定native narrow；不是所有长度都在线自由搜索。 | [src/backends/metal/width_policy.hpp](../../../inferences/tqLLM/src/backends/metal/width_policy.hpp) |
| Lookup draft 与缩小 proposal head | SuffixCopy 只索引 committed token，重复片段提供 proposal；draft_vocab 可裁减 drafter 输出头。 | 只降低 proposal 成本，target 必须验证；缩词表不能直接替换 target 的完整分布。 | [src/spec/suffix_copy.hpp](../../../inferences/tqLLM/src/spec/suffix_copy.hpp)、[include/tqllm/options.hpp](../../../inferences/tqLLM/include/tqllm/options.hpp) |
| Inkling greedy 误差界筛选输出头 | INT8 head 先筛，按每 row 误差上界找仍可能获胜的候选，BF16 rescore，再 device argmax。 | 限该 greedy 实现；sampled 请求按所读 changelog仍走完整logits，不能把普通top-k heuristic都称严格exact。 | [src/models/inkling2/loop/inkling2_kernels.cu](../../../inferences/tqLLM/src/models/inkling2/loop/inkling2_kernels.cu)、[CHANGELOG.md](../../../inferences/tqLLM/CHANGELOG.md) |
| 多请求单执行线程与宽度预算 | HTTP线程排队，单engine线程拥有模型调用；合批验证宽度/每序列draft数受device成本、slots和distinct expert容量限制。 | 并发提高聚合吞吐不保证降低单请求延迟；超容量可退成slot逐个执行。 | [tools/serve/scheduler.hpp](../../../inferences/tqLLM/tools/serve/scheduler.hpp)、[src/models/qwen/qwen35/qwen35_rules.hpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_rules.hpp)、[src/models/qwen/qwen4exp/decode_rules.hpp](../../../inferences/tqLLM/src/models/qwen/qwen4exp/decode_rules.hpp) |
| Mac 实测能力与融合回退 | threadgroup specialization 用小输入试运行选可用大小；fused norm/residual/QK-RoPE/SwiGLU保持舍入物化点，不通过则回MLX链。 | M1 Max 部分kernel只容384线程，不能用统一1024假设；Apple backend使用MLX和自定义Metal，不是完全无MLX。 | [src/backends/metal/threadgroup_fit.hpp](../../../inferences/tqLLM/src/backends/metal/threadgroup_fit.hpp)、[src/backends/metal/prefill_fused.hpp](../../../inferences/tqLLM/src/backends/metal/prefill_fused.hpp) |
| Mac ANE/GPU FFN成本规划 | AneCost计入pack、ANE padded rows、固定提交、GPU share与requantize；ane_plan 仅在预测快于GPU-only时拆分，可在空档预取下层权重。 | 成本规划器存在不等于全部模型/Apple芯片都完成真机覆盖；编译、ANE program和入口仍有条件。 | [src/backends/metal/ane_plan.hpp](../../../inferences/tqLLM/src/backends/metal/ane_plan.hpp) |
| 多设备减少通信和按缓存路由 | layer split 单请求将head/draft/sample放末rank，仅回传picks；多box router按队列、prefill/decode速率与prefix/state transfer成本选择。 | 模型parallel和请求routing是两类机制；router系数是特定实验的heuristic，消费PC链路要另测。 | [src/models/qwen/qwen35/qwen35_rules.hpp](../../../inferences/tqLLM/src/models/qwen/qwen35/qwen35_rules.hpp)、[tools/serve/router.hpp](../../../inferences/tqLLM/tools/serve/router.hpp) |
| 运行时LoRA与图兼容参数 | LoRA scale 放device变量使捕获图读取当前值；A×x分slice固定顺序累加，无atomic，scale0不改输出。 | 此kernel rank<=64、cols偶数；图可读新scale不等于缓存条件可无条件沿用。 | [src/formats/weight/lora.cuh](../../../inferences/tqLLM/src/formats/weight/lora.cuh) |

## 数值精度：三个“exact”不能混在一起

1. **格式质量**：TQ/TQT/INT8/FP8/KV压缩相对原始BF16的误差和任务质量。
2. **执行一致性**：对同一量化模型、相同配置，单token、宽verify、不同batch/chunk/CPU-GPU路径是否给出相同结果。
3. **配置保持**：内存fit是否仍使用用户比较时的KV编码、上下文和prefill分块。

`EngineOptions.exact=true` 默认偏向参考计算精度，却允许显式近似选项获胜并警告；`fit_lossy=true` 还能为装得下自动更改设置并公告。MiniMax/GLM在当前options实现中被标为没有exact路径。若做严格A/B，应记录最终生效配置，并考虑 `-fitl off`，不能只记录启动时的 `--exact on`。

TqCache“原codec的host搬运”、TQ低位KV、`kv-compact`的stand-in，以及QSA稀疏选取是不同机制：前者旨在保持原编码/计算，后几者改变数值/attention集合。代码中的tree==plain也不证明QSA==原dense模型。

## 当前提交的三个重要教训

**首次投机请求也必须更新placement。** 1.8.3修复Inkling只在plain步骤重规划专家的问题。ReplanClock现在在plain和verify路径都调用，每32步起步，稳定后退避至最多1024步。只有热请求快、第一条慢的差异，可能来自策略没有接到全部执行入口，而非kernel没预热。

**数值不同的kernel不能由测速噪声随机选。** CHANGELOG记载Inkling dense kernel固定到Tensor Core族，避免两族耗时接近时重启改变文本。TQ3 matvec规则表也明确不同族舍入不同，固定按device/shape分派。可安全在线测速的候选应先定义等价性；否则选中结果必须进入运行配置和缓存身份。

**文档和测试注释可能落后于实现。** `tests/prefill_path.cpp`仍保留“TQ4没有native GEMM”的历史测试情景；当前`tq4.cpp::has_gemm()`与`tq4_gemm_h()`已经存在。该测试用于旧能力条件的分派验证，不能用旧注释判断当前功能。同理，AGENTS列出的Flash-Next统一KV/format迁移例外，不能被架构示意图掩盖。

上述速度改进来自项目CHANGELOG/注释的报告，本轮没有复现，不据此给出相对其他引擎的排名。

## 与已有项目的比较

| 方向 | tqLLM的实现细节 | 与其他项目的关系 |
| --- | --- | --- |
| 量化格式与内核 | TQ4/TQT同4.125bpw的不同codec；离线Hessian校准；native/dequant成本交点 | 补充Sync-Strata/YASAKANI的格式协同；名称相似不代表比特流可互换 |
| 显存适配 | 明确fit_lossy、最终配置告知、WSL/UMA计数和reload成本 | 补充Strata/XeStrata/TensorFold的预算策略；容量达标必须带配置说明 |
| 异步缓存 | host block epoch、GPU/host压力下去重、bounded预测park | 补充XeMaya/QuixiInfer的生命周期协议；压缩数据可准备一次、多轮复用 |
| 动态策略 | 验证路径也计入驻留重规划，稳定后退避；投机只按净tokens/time选 | 与Strata controller/Splash标定同方向，更强调首轮、JIT和维护成本 |
| 图与精度 | graph owner ID、LRU floor、数值不同kernel不由噪声切换 | 补充ComfyUI图signature/失效规划，缓存身份不只是一块地址 |

## 面向ComfyUI的实验候选

| 顺序 | 最小改动思路 | 应记录的证据 |
| --- | --- | --- |
| 1 | 为一个目标DiT的Linear按格式/形状比较压缩native GEMM与反量化+成熟BLAS，保留实测交点 | M/N/K、格式/真实bpw、反量化固定成本、scratch峰值、整体每步/每图耗时及质量 |
| 2 | 记录显存策略的最终生效参数，默认保持用户图像尺寸/步数/精度；有损退让另列实验 | 是否换精度/分块/模型、显存预算、driver/RAM/pinned、原始和最终配置 |
| 3 | 参考host tier保留已经准备好的块，比较“保留副本省D2H”与“去重增加有效容量” | host复制、D2H/H2D、RAM压力、首轮和热轮，缓存维护是否抵消收益 |
| 4 | 对照当前ComfyUI图捕获，补充shape/owner/权重版本身份、容量上限和热工作集策略 | capture/replay/eviction/graph-break、额外占用、尺寸/LoRA切换及取消后的正确性 |
| 5 | 检查所有模型forward入口是否使用同一资源策略，并以退避限制维护成本 | 首图、普通/控制分支、缓存miss、并发与连续请求，禁止只测预热后的路径 |

Dense DiT的层序是确定的，应优先确定性预取；不需要照搬“猜下一层MoE专家”。KV host-tail/compact、vocab筛选和copy draft主要服务LLM；TQT校准与低位推理若迁移DiT，要重新采集图像/噪声timestep下的激活并验证脸、衣服文字/图案、颜色和跨步误差。

LoRA的device scale可被graph读取，是参数更新的一种方式；它不能替代ComfyUI的条件缓存失效、ModelPatcher生命周期或任意rank/adapter类型的兼容。

## 阅读范围与待跟踪

本轮沿CUDA dense加载/forward/fit、WeightFormat与TQ/TQT、KV store与codec、Flash-Next专家cache/prefill/路由、Inkling residency/spec、Metal width/threadgroup/ANE、serve scheduler/router阅读。查阅的单测为静态理解，没有执行；没有逐行审计所有389个src文件或全部模型、DSpine训练、vision、Metal内嵌shader和第三方代码。

后续重点：格式与实际入口能力、`exact`冲突的实际配置、Flash-Next统一KV/format例外是否消除、TQT DiT量化的可行性、首请求与热请求差异、图缓存是否因低内存反复重建，以及不同硬件的native/dequant交点。

- [跨项目方法索引](../METHODS.md)
- [ComfyUI迁移清单](../COMFYUI.md)
- [统一版本快照](../SNAPSHOTS.md)
- [源码定位与固定版本链接](../evidence.csv)
- [源码文件及哈希清单](../source-files.csv)
