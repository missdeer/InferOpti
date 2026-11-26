# Git 版本快照

15个推理项目、Sync-Strata内嵌engine及ComfyUI基线统一列在下表。完整commit对应研究时实际阅读的本地checkout，未fetch/pull；每条记录保留原始采集时间，不代表远端最新。

| 目录 | 完整HEAD commit | 提交时间 | 采集时间UTC | 工作区 |
| --- | --- | --- | --- | --- |
| `ComfyUI/ComfyUI` | `a277b5bf213e8a44fa34817b7770b54f86ef4cea` | 2026-11-18T16:41:27-05:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/basalt` | `5b8a82c6b7e1069adbce42854e0732b2b19e4f32` | 2026-11-25T13:44:41-07:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/Cheetah` | `e3451a52872c95e8c3af0d347c20dbd55004c6fa` | 2026-11-25T13:18:48+08:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/kinetic` | `1b671b9eea0db10c1d22c51bf524d496d6c6736c` | 2026-11-24T12:35:28+01:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/Maya` | `53885cef8faf0d3b3f7e28c0b1537a056c1f3ecf` | 2026-11-25T21:45:25+00:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/QuixiInfer` | `dc5dbc1bb1cf7a31f6e8a240f7a34bdf2014268b` | 2026-11-20T19:13:34-05:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/RustingStrata` | `e50ec2cc0ecf802319623685ef41d2101d19561b` | 2026-11-26T09:45:19+01:00 | 2026-11-26T09:36:12.272371+00:00 | tracked clean; untracked 0 |
| `inferences/Speedy` | `1f172963a0a483672381e74254b283df22c25e81` | 2026-11-24T07:53:23+02:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/splash` | `d40c250ff248938edebdb91fb6124514e32daeaa` | 2026-11-24T17:28:00-08:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/Strata` | `c234774038d1c9249cb33637d27b895d47b39786` | 2026-11-25T23:59:41+01:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/sync-strata` | `22b5db9ddd60768fab3a33cb19fbde36084e2dec` | 2026-11-21T23:36:23+09:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/sync-strata/engine` | `824dc0a7811456465bb9864c676381c2d7c72b5f` | 2026-11-21T23:20:34+09:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/TensorFold` | `84d9b6ba1367eddd4f97655185a597b5f909be6c` | 2026-11-26T03:17:58+00:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/tqLLM` | `7df840632cd3f108de459e0944b9b6d115532689` | 2026-11-25T22:53:14+00:00 | 2026-11-26T06:18:35.650042+00:00 | tracked clean; untracked 0 |
| `inferences/XeMaya` | `8fa4141666e10d81bf932ab4625aa52a37eed4bd` | 2026-11-12T04:40:11+00:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/XeStrata` | `aadd4f9fea9b5b319855826f6a33c72dce5b529e` | 2026-11-25T22:51:50+09:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |
| `inferences/yasakani` | `631fdfd01f071bdb9131e26da8e06d5ddc39b2a8` | 2026-11-25T13:58:41+09:00 | 2026-11-26T05:44:38.927362+00:00 | tracked clean; untracked 0 |

## 定位信息

- **HEAD**：表中的实际阅读版本，项目笔记与[evidence.csv](evidence.csv)的固定链接使用同一commit。
- **父仓库gitlink**：保存在[snapshots.json](snapshots.json)的`parent_history`和相应子模块记录中；研究时所有已核对gitlink与checkout HEAD一致。
- **文件内容**：[source-files.csv](source-files.csv)统一保存22,181个文件的大小和SHA-256；未提交改动不会进入HEAD，文件清单也不代表全部逐行审计。

## 父仓库采集记录

父仓库观察单独作为版本溯源元数据保存，不计入推理项目数；保留当时的工作区状态，避免把较晚的采集时间套到先前源码上。

| 采集时间UTC | 父仓库HEAD | gitlink记录范围 |
| --- | --- | --- |
| 2026-11-26T05:44:38.927362+00:00 | `b8e3baaa6f686ae5381c728794ea7f010f7fc470` | .agents、inferences/Cheetah、inferences/Maya、inferences/QuixiInfer、inferences/Speedy、inferences/Strata、inferences/TensorFold、inferences/XeMaya、inferences/XeStrata、inferences/basalt、inferences/kinetic、inferences/splash、inferences/sync-strata、inferences/yasakani |
| 2026-11-26T06:18:35.650042+00:00 | `adfa2ae9a4c04fedbb66cce5a9bbff82f3c635bb` | inferences/tqLLM |

| 2026-11-26T09:36:12.272371+00:00 | `5a68ee5199a4413aa179a0c749be8a02db56d507` | 全部已登记gitlink，含RustingStrata；逐项记录见snapshots.json |

这些父仓库快照在研究文档写入前采集，工作区状态并非当前状态。没有更改项目源码、子模块版本或Git全局配置。历史文档组织可通过Git提交查阅，统一JSON/CSV作为当前查询入口。

## 定位示例

```powershell
# 查看固定版本源码，不切换工作区
git -C inferences/Strata show c234774038d1c9249cb33637d27b895d47b39786:engine/src/core/expert_cache.cpp
git -C inferences/tqLLM show 7df840632cd3f108de459e0944b9b6d115532689:src/models/qwen/qwen35/prefill_path.hpp

# 内嵌engine使用自己的commit
git -C inferences/sync-strata/engine show 824dc0a7811456465bb9864c676381c2d7c72b5f:src/kernels/cpu/tq_cpu.cpp
```

重新获取代码时还需要初始化嵌套子模块；不要为了查阅历史覆盖已有未提交工作。统一[evidence.csv](evidence.csv)共有265个证据条目，包含固定commit的远端链接。
