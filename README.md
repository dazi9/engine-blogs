engine-blogs
============

深度技术博客系列：实现两个存储引擎（Bitcask / B+Tree+LSM）的完整复盘。

- 01_bitcask_deep_dive.md — append-only、Hint 文件与 Compaction 的设计取舍
- 02_btree_vs_lsm.md — 同一套缓冲池上的两个引擎对比（真实实验数据）
- 03_papers_to_code.md — 从 SIGMOD 论文（LRU-K、Monkey）到 C++ 代码
- 04_fuzzing_battle_report.md — 模糊测试抓到 12 个真实 bug 的战报
- fuzz-brief.md — 第 4 篇的素材底稿

对应的项目仓库：pybitcask、hybriddb-sys。
