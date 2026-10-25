# 模糊测试战报素材（供 04 号文章使用）

以下是本轮 fuzzing 工厂在两个引擎上抓到并修复的 12 个真实 bug 摘要。
所有细节以两个仓库 hardening 分支的提交记录为准：
- pybitcask: `git log` → "fix: crash-consistency hardening from model-based fuzzing"
- hybriddb-sys: `git log` → 两个 fix 提交

## pybitcask（4 个，全部由五 profile 模糊测试器抓到）

1. **过期 hint 遮蔽已落盘数据（数据丢失）**：重启时只要 hint 文件存在就无条件信任；
   正常关闭后进程崩溃，崩溃前已刷入 OS 的写入全部隐形。
   修复：`_hint_covers_file()` 校验 hint 覆盖范围 == 数据文件大小，不匹配回退全量扫描。
2. **删除的 key 重启后复活**：hint 只记活 key，tombstone 进不了恢复路径——
   put k（段2 封口 hint 含 k）→ delete k（tombstone 在段3）→ 重启时段2 hint 把 k 加回来。
   修复：KeyDir 按段记录 tombstone 元数据，hint 写入时带上（FLAG_TOMBSTONE）。
3. **恢复回放顺序颠倒**：compaction 把旧快照写进比 active 更高编号的段文件，
   按文件序回放时旧值反向覆盖新值。修复：KeyDir 冲突按记录时间戳裁决。
4. **同毫秒 delete→re-put 顺序错乱**：毫秒级墙上时钟分不出先后，
   tombstone 和 re-put 时间戳相等时删除误胜。修复：逻辑时钟
   `max(last+1, wall_ms)`，重启时从回放数据播种。

## hybriddb-sys（8 个）

1. **WAL lsn 跨会话归零（数据丢失）**：WALManager 每次构造 lsn 从 1 开始，
   但 meta 页持久化了上一会话的大 checkpoint_lsn_，恢复时新会话记录全部被
   `rec.lsn <= checkpoint_lsn_` 跳过。修复：打开时播种 `checkpoint_lsn_ + 1`。
2. **steal/no-steal 策略错配（最深的 bug）**：缓冲池逐出时单独刷脏页（盘上半新半旧），
   恢复却假设盘上=完整检查点状态 → 崩溃后回放到残破树上，树导航断裂、
   几十个 key 丢失、put 报 "Could not locate leaf page"。
   修复：no-steal 缓冲池（脏页只在检查点落盘）+ 脏页压力主动 checkpoint + 重试。
3. **首次检查点前崩溃**：盘上只有 FREE 空白页骨架，恢复当真树导航。
   修复：`checkpoint_lsn_ == 0` 时从空根重建全量回放。
4. **meta 页先于树页落盘**：根分裂时 meta 立即 flush，崩溃后 meta 指向从未写盘的页。
   修复：只有新建库和 checkpoint 才立即持久化 meta。
5. **LSM MANIFEST 无校验**：manifest 是唯一真相源却是无校验文本，损坏后静默丢
   SST 条目 → 读出旧值。修复：CRC 头 + 原子发布（tmp+fsync+rename+目录fsync）。
6. **SST 数据页损坏 → 静默旧读（最危险）**：load_manifest 重建 fence pointers
   遇 CRC 失败就 break，布隆基于残缺 key 集构建，查询静默跳过该 SST。
   修复：任何页损坏或文件缺失直接拒绝打开（构造函数抛异常 fail-closed）。
7. **compaction 先删文件后发 manifest** / **flush 先截断 WAL 后发 manifest**：
   中间崩溃丢数据。修复：先原子发布 manifest 再删文件/截断 WAL + 启动 GC 孤儿 SST。
8. **只读句柄 close 也 fsync**：SST 点查每次 open/close 白付一次 fsync。
   修复：只有写过数据的句柄才 fsync（点查路径预计 5-10 倍提升）。

## 验证强度

- pybitcask：52 单测全绿 + 529 轮 fuzz（3 种子）0 findings
- hybriddb-sys：12 套测试全绿 + 终验 110 轮（ASan/UBSan）0 findings
- 面试叙事角度：每个 bug 都是一条真实的失效路径分析；"steal/no-steal 错配"
  和"静默旧读"是能讲透 ARIES 前提条件和 LSM 可靠性设计的好故事。
