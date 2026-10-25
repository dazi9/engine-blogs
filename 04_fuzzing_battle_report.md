# 我给自己的存储引擎做模糊测试，抓到了这些 bug

> 这一篇是战报，不是教程：这一轮 hardening，我给系列前两篇的两个引擎——pybitcask（Python KV 存储）和 hybriddb-sys（C++ 双引擎）——跑了模型模糊测试和故障注入，抓出并修掉了 **12 个真实 bug**（pybitcask 4 个、hybriddb-sys 8 个）。每个 bug 我都写了回归测试钉住。下面按"怎么抓的、坏在哪、怎么修的"复盘。行号对应两个仓库 hardening 分支。

先给结论定个性：这 12 个 bug **没有一个**是"算法写错了"——没有一次是 B+ 树分裂切错半、布隆算错位。每一个都是**两个组件对"此刻谁拥有真相"的分歧**：hint 文件以为自己是真相、WAL 计数器以为自己是真相、MANIFEST 以为自己是真相、缓冲池以为盘上是什么状态。算法错误我在写单元测试时就能抓住；缝上的错误只有让两个组件在崩溃里对质才能暴露。

## 一、fuzzer 长什么样：五份契约，一个预言机

pybitcask 的 fuzzer（fuzz/model_fuzz.py）是模型测试：内存里维护一个 Python dict 当 oracle，引擎每做一步就和 oracle 对账。核心是 `full_verify`：比对 key 集合、长度、然后逐 key 比值（model_fuzz.py:122-141）——任何一处不齐就往 findings.jsonl 写一条 finding。

比 oracle 更重要的是**契约**。每个 profile 开头用注释写明"这个故障注入之后，什么性质必须成立"（model_fuzz.py:4-22）：

- **ops**：随机 op 序列（put/delete/get/scan/batch/compact/reopen），每步对账；
- **torn**：干净关闭 → 把 active 段尾部截掉 1~512 字节 → 重开。契约：要么启动失败且报错干净，要么成功且截断前的每个 key 都可读；
- **corrupt**：干净关闭 → 随机段文件里翻转 1~3 个 bit → 重开。契约：**绝不返回错的数据**——要么报错，要么返回对的值；
- **kill**：子进程写两个会话，第二个会话中途 `os._exit(1)` 硬崩（kill_writer.py:70）。契约：已刷进 OS 页缓存的写入，除最后 2 条外必须全部存活；
- **ttl**：任何公开 API 都不许把过期 key 当活的，重开和 compaction 之后也一样。

kill profile 单独拉一个子进程是有原因的：进程内的"假崩"要绕过析构函数、锁清理、hint 刷新，伪装它们比真的死掉更难写对。子进程里 `os._exit(1)` 跳过一切清理，是最诚实的崩溃。

故障注入原语是引擎自己带的。hybriddb-sys 的 B+ 树引擎有 `simulate_crash_without_flush()`（btree.cpp:492-497）——丢弃缓冲池全部脏帧、只留 WAL，精确模拟"断电时页缓存消失"；WAL 管理器有 `inject_torn_tail_bytes` 往日志尾部补垃圾字节（wal.cpp:199-208）。**能被测的前提是能被注入**——这两个函数不是测试工具，是引擎 API 的一部分。

还有个当时随手做的决定后来立了功：key 池里掺了中文（`用户:{i}`、汉字 value 字母表，model_fuzz.py:66-72）。第一篇里写过的"中文 key 一写就坏"就是上游项目掉过的坑，这次我把池子做成中英混合，等于每个 profile 都在顺便回归这个坑。

## 二、pybitcask 的 4 个 bug：hint 文件教我做人

四个 bug 都和崩溃恢复的"重放"有关，第一篇里有完整版本，这里只给战报摘要：

1. **过期 hint 遮蔽已落盘数据**：恢复时无条件信任 hint 文件，干净关闭后硬崩，崩溃前已进 OS 页缓存的写入全部隐形。修复：`_hint_covers_file` 校验 hint 覆盖的字节范围恰好等于数据文件大小，不等就退回全量扫描（segment.py:250-280）。
2. **被删的 key 复活**：hint 只记活 key，墓碑进不了恢复路径——段 1 的 hint 把 k 加回来，段 2 的墓碑没人重放。修复：KeyDir 按段登记墓碑元数据，hint 写入时带上（keydir.py:33、96-103，segment.py:483-494）。
3. **回放顺序颠倒**：compaction 产物拿比 active 段更高的文件号，按文件序重放时旧快照反向覆盖新值。修复：冲突按记录时间戳裁决，旧的不许覆盖新的（keydir.py:60-61）。
4. **同毫秒时序错乱**：毫秒墙上时钟分不出 delete 和紧跟的 re-put 谁先。修复：逻辑写时钟 `max(last+1, now_ms)`，重启时用恢复出的最大时间戳播种（engine.py:269，keydir.py:219-233）。

四个 bug 连起来的教训一句话：**恢复的正确性取决于"重放所有来源能否收敛到同一状态"，任何一个来源少带一点信息，收敛就碎**。hint 是摘要，摘要必须自证完整、必须携带全部状态变更——包括删除。

## 三、hybriddb-sys 的 8 个 bug：按缝分类

C++ 引擎这边 8 个 bug，我按它们断裂的那条缝分成四组。

### 3.1 缝一："盘上到底是什么状态"——两个最深的问题

**Bug 1：WAL lsn 跨会话归零。** WALManager 的计数器成员默认值是 `next_lsn_{1}`（wal.hpp:77）——每个新会话从 1 重新计数，没人觉得这有问题，直到 meta 页开始持久化上一会话的 checkpoint 水位。场景：会话 A 写 100 条、checkpoint（水位 ~100+）、正常关闭；会话 B 重开、写 50 条、硬崩。会话 B 的 lsn 是 1~50，全部小于水位，恢复时被 `if (rec.lsn <= checkpoint_lsn_) continue;`（btree.cpp:157）**整段跳过**——崩溃前明明落了 WAL 的 50 条写入静默蒸发。这个 bug 的可怕在于它需要"跨会话"才触发，单会话内的所有测试永远碰不到它。修复一行：打开引擎时播种 `checkpoint_lsn_ + 1`（btree.cpp:132-136）。回归测试用三个会话钉住它：A 写 100 条 checkpoint，B 写 50 条硬崩，C 恢复后断言 150 条全在且值按会话区分（tests/test_main.cpp:264-310，`test_wal_lsn_continuity_across_sessions`）。

**Bug 2：steal/no-steal 策略错配。** 这是我复盘时最想重做的一条，第二篇里详细写过，这里只补 fuzzer 视角。症状：kill/torn 类崩溃注入后恢复，get 丢几十个 key，put 报 `Could not locate leaf page`（btree.cpp:295）。根因：缓冲池逐出脏页时单独刷盘（buffer_pool.cpp:271-274），崩溃后盘上各页处于不同 lsn 的混合状态，而恢复协议假设"盘上 = 完整检查点镜像"，只回放增量——假设被 steal 悄悄破坏了。修复是三件套：no-steal（脏帧拒绝逐出并上报耗尽，buffer_pool.cpp:262-270）+ 脏帧压力主动 checkpoint（btree.cpp:482-490）+ 耗尽后重试（btree.cpp:380-386）。回归测试把缓冲池压到 **6 帧**（tests/test_main.cpp:341-377，`test_btree_dirty_pressure_small_pool`）——6 帧 << 树的页数，强制一整场会话里反复触发压力检查点，然后硬崩、恢复、逐 key 断言。

这一组 bug 教我的是：**恢复协议不是一段代码，是一组前提**。"盘上是什么"这个问题的答案必须有人保证，而保证它的代码（缓冲池策略）和依赖它的代码（恢复）相隔两个文件——缝就是这么来的。

### 3.2 缝二："谁先落盘"——两个顺序 bug

**Bug 3：首次检查点前崩溃。** 建库时 allocate_page 会给每个页写 FREE 类型的空白骨架（disk_manager.cpp:106-121），第一次 checkpoint 之前崩溃，盘上就是一副空白骨架加一个指向它的 meta。旧恢复逻辑拿骨架当真树导航，80 条写入全丢。修复：`checkpoint_lsn_ == 0` 时把根重置成空叶、从零全量回放（btree.cpp:141-149）。回归测试 `test_btree_crash_before_first_checkpoint`（tests/test_main.cpp:312-339）。

**Bug 4：meta 页先于树页落盘。** 根分裂时 meta（含新根页号）立即 flush，崩溃后 meta 指向一个从未写盘的页。修复：把"立即持久化"收窄成一个显式参数，只有新建库和 checkpoint 允许立即刷 meta（btree.cpp:69-89 的 `persist_immediately`，81-84 行的注释就是给这个 bug 的墓志铭），根分裂路径改成只更新内存（btree.cpp:223）。

和 Bug 7 的两处 LSM 顺序问题（见下）合起来看，这一组的通用形态是：**崩溃后必须存在的依赖关系（A 依赖 B），落盘顺序必须反过来（B 先于 A）**。checkpoint 里 flush → 水位 → meta → 截 WAL 的顺序（btree.cpp:474-480），每一格都是被 fuzzer 逼出来的。

### 3.3 缝三："损坏了要不要继续"——fail-closed 的确立

**Bug 5：MANIFEST 无校验。** MANIFEST 是 LSM 唯一的真相源（哪些 SST 活着），却是无校验的纯文本。一个 bit 翻转后旧版照样加载，静默丢掉部分 SST 条目——**读出来的是旧值，还不报错**。修复两层：文件加 `"MANIFEST1 " + CRC` 头、写入走 tmp + fsync + rename + 目录 fsync 的原子发布（lsm.cpp:636-680）；加载时校验失败直接抛 Corruption 拒绝打开（lsm.cpp:695-707）。回归测试在 MANIFEST 中间翻一个 bit，断言构造函数必须抛异常（tests/test_main.cpp:379-419）。

**Bug 6：SST 数据页损坏 → 静默旧读。** 这是我评的"最危险"：load_manifest 重建 fence pointer 和布隆时，遇到 CRC 失败的页就 break——布隆基于**残缺的 key 集合**重建，查询时静默跳过受损 SST 里实际存在的 key，读到更老的版本。静默、可复现、且所有单测全绿。修复：任何页损坏或 manifest 引用的文件缺失，直接拒绝打开（lsm.cpp:739-742、720-722）。回归测试翻 SST 里一个 bit，断言打不开（tests/test_main.cpp:421-469）。

这两个 bug 让我把 fail-closed 从直觉升级成纪律：**对存储引擎来说，"拒绝服务"和"返回错数据"之间没有等价交换——后者永远更贵**。旧读不会崩、不会报错、不会进监控，用户只在某天发现订单金额不对。8 个修复里 3 个把路径改成了"构造函数抛异常"，全是有意的。

### 3.4 缝四："顺序对了，然后呢"——一个性能 bug

**Bug 8：只读句柄 close 也 fsync。** LSM 点查的路径是每个 SST 开一个 DiskManager（lsm.cpp:308），用完就关——而 close 无条件 fsync（disk_manager.cpp:29-40 修复前的行为）。一次点查扫三层，白付三次 fsync。这个 bug 和正确性无关，fuzzer 的契约也照单全收（fsync 不改变可见状态），它是 8 个里唯一的性能问题，也是唯一一个"每天都在收税"的。修复：只有写过数据的句柄才 fsync（`has_written_` 标记，disk_manager.cpp:29-40；page.hpp:251-252 的注释）。

### 3.5 顺带：两处持久化顺序（Bug 7）

LSM 侧还有一对顺序 bug：flush 原来先清 memtable、截 WAL，再存 manifest——中间崩溃，可恢复副本没了而新 SST 不在册，数据蒸发；compaction 原来先删旧文件再发 manifest——中间崩溃，旧 manifest 指向不存在的文件。修复都是同一个模式：**先原子发布 manifest，再销毁可恢复副本 / 旧文件**（flush：lsm.cpp:373-380，注释写明这个不变量；compaction：先捕获将被取代的文件列表 456-458，发布 manifest 475，最后才删 478-481），启动时再 GC 掉孤儿 SST（lsm.cpp:763-771）。

## 四、修完之后：验证强度，和我现在信什么

修复后的验证强度（来自 fuzz 战报记录）：pybitcask 52 个单元测试全绿 + 三个种子共 529 轮 fuzz 零 finding；hybriddb-sys 12 个测试套件全绿（tests/test_main.cpp:579-590）+ 终验 110 轮（ASan/UBSan 打开）零 finding。WAL 崩溃恢复的专项实验（results/exp4_wal_crash_recovery.csv）四种故障场景全部 PASS：1000 条未刷盘事务 1.196 ms 重放完毕，注入的 13 字节 WAL 撕裂尾被精确截断。

另外有一个**没有**算 bug 的 finding 值得一提：ttl profile 报过一条 `TTL-EARLY-GONE`（findings.jsonl 里现在还留着这条，seed 300153）——5 秒 TTL 的 key "提前"死了。复跑十几轮都不再现，最合理的解释是 fuzz 进程在高负载机器上被调度器挂起了几秒。它教会我的和那 12 个一样多：**断言依赖什么时钟假设，要写在 profile 的注释里**（model_fuzz.py:388-389 现在就有这句）。

12 个 bug 过完，我现在做存储代码时信四件事：

1. **先写契约再写注入**。五个 profile 每个开头那段"故障之后什么必须成立"的注释，比 fuzzer 本身值钱——没有契约的 fuzzing 只是崩溃检测，有契约的 fuzzing 才是正确性证明的搜索；
2. **崩溃注入比数据注入高产**。12 个 bug 里 9 个是崩溃/重放类（hint 遮蔽、墓碑复活、回放顺序、lsn 归零、steal 错配、首次检查点、meta 顺序、两处 LSM 顺序），bit 翻转类只贡献了 fail-closed 那 2 个。存储引擎的复杂性长在"跨过一次崩溃后状态如何收敛"上，不长在字节里；
3. **每个修复钉一个以场景命名的回归测试**。hybriddb-sys 的 12 个套件里，5 个是这批 bug 的直接钉子（lsn 跨会话连续性、首次检查点、6 帧脏页压力、MANIFEST 损坏 fail-closed、SST 损坏 fail-closed），另有一个 WAL 故障注入套件覆盖撕裂尾场景（tests/test_main.cpp:122）——回归测试不是验证修复，是防止三个月后的我自己把它改回去；
4. **fail-closed 是对抗"静默错误数据"的唯一原则**。12 个 bug 里最危险的三个（静默旧读、MANIFEST 静默丢条目、过期 hint 遮蔽数据）共同点是不崩溃、不报错、监控全绿。修它们的全部思路就一句话：有疑问，拒绝启动。
