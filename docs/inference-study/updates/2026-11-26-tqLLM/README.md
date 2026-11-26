# tqLLM增量归档：2026-11-26

- 项目HEAD：`7df840632cd3f108de459e0944b9b6d115532689`；提交时间：`2026-11-25T22:53:14+00:00`；main；tracked clean、untracked 0。
- 父仓库本次HEAD：`adfa2ae9a4c04fedbb66cce5a9bbff82f3c635bb`；索引gitlink与上述tqLLM HEAD一致。
- 采集时间UTC：`2026-11-26T06:18:35.650042+00:00`。未fetch/pull，所记当前版本仅指本地checkout。
- [学习笔记](../../projects/tqLLM.md)
- [snapshots.json](snapshots.json)：项目/父仓库本次版本与工作区状态，原13项目快照保持原样。
- [evidence.csv](evidence.csv)：61个源码/说明证据条目，固定commit链接。
- [source-files.csv](source-files.csv)：825个源/测试/工具文件的路径、大小和SHA-256。
- [inventory-summary.json](inventory-summary.json)：计数与排除规则。

本次清单为源码395、测试/bench342、工具87、归档旧源码1。增加`.inc`扩展名，因为实际CUDA kernel使用include片段；`archive/`文件只为定位保留，不认定在构建中使用。文件计数不等于已完成每文件逐行阅读。

[原始13项目快照](../../snapshots.json)、[原始文件清单](../../source-files.csv)与[evidence](../../evidence.csv)没有覆盖。现在累计14个项目，文件清单共21,913条（本次含`.inc`、初次扩展名范围不同），源码证据共223条。并未重新核读或更新此前13个项目的版本。

定位示例（不切换工作区）：

```powershell
git -C inferences/tqLLM show 7df840632cd3f108de459e0944b9b6d115532689:src/models/qwen/qwen35/prefill_path.hpp
```

本次未构建、未下载模型、未跑测试或硬件基准；所有项目性能数据仍是源码/文档自报。
