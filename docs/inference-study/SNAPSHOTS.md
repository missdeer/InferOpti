# Git 版本快照

以下主表保留初次13项目的历史快照。新增tqLLM见本页末尾的增量；机器可读快照也单独保存，未覆盖原文件。

采集时间（UTC）：`2026-11-26T05:44:38.927362+00:00`。只读取本地 checkout，未 fetch/pull；这里的“当前”不代表远端最新。

13 个项目均为 tracked clean、untracked 0。Sync-Strata 的 engine 单独列出。本轮所有推理源码 gitlink 均与父仓库 index 记录一致。

| 目录 | 完整 HEAD commit | 提交时间 | 工作区 |
| --- | --- | --- | --- |
| `.` | `b8e3baaa6f686ae5381c728794ea7f010f7fc470` | 2026-11-26T12:03:05+08:00 | tracked clean; untracked 6 |
| `inferences/basalt` | `5b8a82c6b7e1069adbce42854e0732b2b19e4f32` | 2026-11-25T13:44:41-07:00 | tracked clean; untracked 0 |
| `inferences/Cheetah` | `e3451a52872c95e8c3af0d347c20dbd55004c6fa` | 2026-11-25T13:18:48+08:00 | tracked clean; untracked 0 |
| `inferences/kinetic` | `1b671b9eea0db10c1d22c51bf524d496d6c6736c` | 2026-11-24T12:35:28+01:00 | tracked clean; untracked 0 |
| `inferences/Maya` | `53885cef8faf0d3b3f7e28c0b1537a056c1f3ecf` | 2026-11-25T21:45:25+00:00 | tracked clean; untracked 0 |
| `inferences/QuixiInfer` | `dc5dbc1bb1cf7a31f6e8a240f7a34bdf2014268b` | 2026-11-20T19:13:34-05:00 | tracked clean; untracked 0 |
| `inferences/Speedy` | `1f172963a0a483672381e74254b283df22c25e81` | 2026-11-24T07:53:23+02:00 | tracked clean; untracked 0 |
| `inferences/splash` | `d40c250ff248938edebdb91fb6124514e32daeaa` | 2026-11-24T17:28:00-08:00 | tracked clean; untracked 0 |
| `inferences/Strata` | `c234774038d1c9249cb33637d27b895d47b39786` | 2026-11-25T23:59:41+01:00 | tracked clean; untracked 0 |
| `inferences/sync-strata` | `22b5db9ddd60768fab3a33cb19fbde36084e2dec` | 2026-11-21T23:36:23+09:00 | tracked clean; untracked 0 |
| `inferences/TensorFold` | `84d9b6ba1367eddd4f97655185a597b5f909be6c` | 2026-11-26T03:17:58+00:00 | tracked clean; untracked 0 |
| `inferences/XeMaya` | `8fa4141666e10d81bf932ab4625aa52a37eed4bd` | 2026-11-12T04:40:11+00:00 | tracked clean; untracked 0 |
| `inferences/XeStrata` | `aadd4f9fea9b5b319855826f6a33c72dce5b529e` | 2026-11-25T22:51:50+09:00 | tracked clean; untracked 0 |
| `inferences/yasakani` | `631fdfd01f071bdb9131e26da8e06d5ddc39b2a8` | 2026-11-25T13:58:41+09:00 | tracked clean; untracked 0 |
| `inferences/sync-strata/engine` | `824dc0a7811456465bb9864c676381c2d7c72b5f` | 2026-11-21T23:20:34+09:00 | tracked clean; untracked 0 |
| `ComfyUI/ComfyUI` | `a277b5bf213e8a44fa34817b7770b54f86ef4cea` | 2026-11-18T16:41:27-05:00 | tracked clean; untracked 0 |

## 三种版本信息不要混用

- **HEAD**：本地实际阅读的提交，以上表为准。
- **父仓库 gitlink**：父仓库索引记录的子模块目标版本，保存在 [snapshots.json](snapshots.json) 的 gitlinks 中。
- **文件内容**：未提交修改不会进入 commit；本次项目源码干净，另有 [source-files.csv](source-files.csv) 为纳入清单的文件记录 SHA-256。清单不代表每个文件都完成了人工逐行阅读。

根仓库原有未跟踪文档/ComfyUI 路径已记录；本快照在写研究资料前采集，因此根仓库 untracked 计数不是交付后的计数。未更改源码、子模块版本或 Git 全局配置。

## 定位示例

```powershell
# 查看本次 Strata 源文件，不切换当前工作区
git -C inferences/Strata show c234774038d1c9249cb33637d27b895d47b39786:engine/src/core/expert_cache.cpp

# Sync-Strata 内核应进入子模块，并用它自己的 commit
git -C inferences/sync-strata/engine show 824dc0a7811456465bb9864c676381c2d7c72b5f:src/kernels/cpu/tq_cpu.cpp
```

如果要从父仓库重新获取这批代码，父仓库提交是 `b8e3baaa6f686ae5381c728794ea7f010f7fc470`，还需要初始化嵌套子模块。不要为了查阅历史而覆盖现有未提交工作。

162 条源码证据在 [evidence.csv](evidence.csv) 中带有固定 commit 的远端 permalink，方便在不依赖当前分支目录布局时查阅。

## 2026-11-26：新增 tqLLM

| 目录 | 完整HEAD commit | 提交时间 | 工作区 |
| --- | --- | --- | --- |
| `inferences/tqLLM` | `7df840632cd3f108de459e0944b9b6d115532689` | 2026-11-25T22:53:14+00:00 | tracked clean; untracked 0 |

本次父仓库HEAD为`adfa2ae9a4c04fedbb66cce5a9bbff82f3c635bb`，索引gitlink与上述tqLLM HEAD一致。完整元数据、825文件哈希和61个证据链接见[tqLLM增量归档](updates/2026-11-26-tqLLM/README.md)。此前13个项目的版本未在这轮重新采样。
