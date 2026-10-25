# 我实现了一个 Bitcask 引擎：append-only、Hint 文件与 Compaction 的设计取舍

> 项目地址：pybitcask（Python，约 3100 行核心代码 + 一套模型模糊测试）。这篇是我把 Riak Bitcask 论文落到代码之后写的复盘，重点讲四个设计取舍，以及我的 fuzzer 抓出来的四个真实 bug。

大二上学期，我跟着 avinassh/py-caskdb 的思路写了一个最简 KV 存储：12 字节的 `<LLL` 头（timestamp、key_size、value_size），append-only 单文件，内存里一个 dict 当索引。能跑，但有两个问题让我睡不着觉：一是**中文 key 一写就坏**——上游把 `len(key)`（字符数）而不是 `len(key.encode('utf-8'))`（字节数）编进头里，"用户:1" 这种 key 读出来直接错位；二是**进程被 kill -9 之后数据库打不开**。所以我重写了整个格式层，起名 Bitcask v2，顺便把 Riak Bitcask 论文里的 hint 文件、merge、多段文件都补齐了。

下面按我实际纠结过的顺序讲。

## 一、为什么是 append-only

Bitcask 的核心决定只有一个：**磁盘上只有顺序追加，永远不原地改**。

- 写路径是纯顺序 I/O。我的 `append_bytes` 只做 seek 到文件尾、write、推进 `write_position`（segment.py:78-112）。
- 崩溃时最坏情况是"最后一条记录写了一半"，损坏永远是尾部前缀，不会出现"B+树页写到一半、父子指针不一致"这种中间状态。这个性质后面救了我很多次。
- 代价是：同一个 key 更新 N 次就有 N 份旧值躺在文件里，删除也只是追加一条墓碑记录。空间要靠后面的 compaction 收。

记录格式我定的是 29 字节定长头 + key + value（format.py:110-119）：

```python
RECORD_HEADER_FORMAT: str = "<IQBQII"   # format.py:118
# crc32(4B) | timestamp(8B) | flags(1B) | expire_at(8B) | key_size(4B) | value_size(4B)
```

flags 用位标记：`0x01` 是墓碑，`0x02` 是带 TTL（format.py:121-123）。CRC32 覆盖"头去掉 CRC 字段之后的所有字节 + key + value"（format.py:154-159），读回时逐条校验（format.py:264-271）。

一个省 I/O 的小设计：KeyEntry 里除了记录起点 `position`，还额外存了 `value_pos`（value 在文件里的绝对偏移）。当用户关掉校验（`verify_checksum=False`）时，读路径可以直接 `os.pread(value_pos, value_size)` 跳过整个头（engine.py:414-415，segment.py:151-167）——一次读只碰 value 字节。开着校验时就没这个便宜了，CRC 必须连头一起算，只能读完整记录。

## 二、KeyDir：全内存索引的代价

Bitcask 的读模型是：内存哈希表 KeyDir 存 `key -> (file_id, position, size)`，get 先查表，再一次磁盘读拿到值。写入和删除都只是更新这张表。

这个设计的账很好算但很疼：**内存占用正比于 key 总数，与 value 大小无关**。我的一条 KeyEntry 有 7 个字段（format.py:33-47），Python 对象再怎么用 `slots=True` 压，一条也要一两百字节。也就是说 1000 万个 key 光索引就吃掉 1-2 GB——这就是论文里明说的"keys must fit in RAM"，我复现的时候才算真正理解这句话的分量。value 可以很大没关系，key 才是索引的计费单位。

KeyDir 还背着两笔额外账：

1. **死字节统计**。每次 put 覆盖旧值、或者墓碑生效，都要把旧记录的 `total_size` 加到对应段的 `_dead_bytes_by_file[file_id]` 上（keydir.py:29，keydir.py:62，keydir.py:95）。这个计数器是 compaction 决定"值不值得搬"的依据。
2. **墓碑登记簿**。`_tombstones_by_file` 记住每条墓碑记录自己在哪个文件哪个位置（keydir.py:33）。这看起来很怪——key 都删了还记它干嘛？这是为了修一个 bug，第二节讲 hint 文件的时候会说。

读路径本身很直白（engine.py:387-418）：查 KeyDir → 查 LRU 缓存 → pread。缓存是 80 行的 `OrderedDict` LRU（cache.py:12-97），有个小细节：超过 `max_value_bytes`（默认 1 MiB）的大 value 不进缓存，免得一个大 blob 把热数据全挤出去（cache.py:55-58）。

## 三、CRC32 与 torn-write 恢复：区分"撕裂"和"损坏"

崩溃恢复是我在这个项目里花时间最多的地方。启动时逐条扫描数据文件（segment.py:331-443），有三类坏法，处理方式完全不同：

1. **头都不完整**（读不满 29 字节）——进程死在写头的过程中。`recover_tail=True` 时直接截断（segment.py:349-360）。
2. **头完整但 payload 不完整**（`offset + total_size > file_size`）——死在写 value 的一半。同样截断（segment.py:366-381）。
3. **CRC 校验失败**——这个最微妙。**只有当它是文件里最后一条记录时才允许截断**（segment.py:396-404）；中间任何一条 CRC 错了，都抛 `CorruptDataError` 直接拒绝启动。

第三条的区别对待是故意的。append-only 文件的物理事实是：坏字节只可能出现在尾部（写到一半就死了），文件中段的字节一旦落盘就没理由变。所以"最后一条 CRC 错"几乎必然是撕裂，可以治；"中间 CRC 错"是磁盘腐坏或者别的什么东西在咬你的文件，属于未知故障，静默截掉半截文件是在掩盖问题。我的原则是 **fail-closed：要么修好，要么拒绝启动，绝不返回错的数据**。

截断本身一行代码：`os.truncate(path, valid_offset)`，然后重开文件、把 `write_position` 拨回去（segment.py:445-454）。

## 四、Hint 文件：加速冷启动，然后被它坑了两次

多段模式下冷启动要重放所有历史段。数据文件越长，重放越慢，而且每个旧 value 都要过一遍内存。hint 文件的思路：**给每个段配一份"索引快照"，只存元数据不存 value**——每条 49 字节定长头 + key（format.py:303-305）：

```python
HINT_HEADER_FORMAT: str = "<IQBQIQIQI"   # format.py:304
# crc32 | timestamp | flags | expire_at | key_size |
# value_pos | value_size | record_pos | record_size
```

带着 `record_pos/record_size`（回数据文件定位用的）和 `value_pos/value_size`（跳过头的快速读用的），恢复时根本不用碰数据文件的 value。我的 `_load_from_hint_file`（segment.py:282-329）只解析 hint 就能把 KeyDir 重建出来。段被封存（rotation）和干净关闭时都会写 hint（engine.py:296-306，engine.py:946-976），写入是"写 .tmp → fsync → rename"的原子替换（segment.py:467-497）。

听起来很美，然后我在模糊测试里连吃了两个 bug。

### Bug 1：过期的 hint 把新写入"遮"没了

第一版 `scan_into_keydir` 的逻辑是：有 hint 就信 hint，无条件。压测用的 kill profile（fuzz/kill_writer.py）复现了丢数据：进程 A 干净 close（此时写了 hint），进程 B 重新打开、继续往**同一个 active 段**追加数据，然后 `os._exit(1)` 硬崩。下次启动，B 写的数据全丢了——因为 hint 还停留在 A 关闭时的状态，而恢复路径信了它，根本没去扫 B 追加的那段文件。

修复是 `_hint_covers_file`（segment.py:250-280）：解析 hint 的每条头（不需要碰 key 和 value，纯 O(记录数)），取出所有 `record_pos + record_size` 的最大覆盖终点，**只有它恰好等于数据文件大小时才信任 hint**：

```python
max_end = max(max_end, record_pos + record_size)   # segment.py:278
...
return max_end == file_size                        # segment.py:280
```

硬崩后文件比 hint 长（追加了没写 hint），torn-tail 截断后文件比 hint 短——两种情况都不等，都会打日志退回全量扫描数据文件（segment.py:229-241）。hint 从"真相源"降级成"只在能自证完整时才可用的加速器"。

### Bug 2：墓碑没进 hint，被删的 key 复活了

第二版我修掉了上面的问题，然后 ops profile 抓到了更阴的一招。场景：段 1 里写入了 `k`，段满了 rotate 到段 2，在段 2 里 delete `k`。段 1 的 hint 是 rotate 时写的，里面带着 `k -> 段1的旧值`；段 2 的墓碑只存在于数据文件里。恢复时按 file id 从小到大重放：段 1 的 hint 把 k 加回 KeyDir，段 2 的墓碑——**根本没被重放，因为它在 hint 里不存在**。被删掉的 key 就这么还魂了。

根因是我想当然地认为"hint 是这段数据文件的摘要，而墓碑不是数据"。但恢复的正确性要求 hint 是**完整的重放替身**：别人重放数据文件能得到什么状态，重放 hint 就得得到什么状态，删除也算状态。修复分两半：

- KeyDir 加了 `_tombstones_by_file` 登记簿（keydir.py:33），delete 时把墓碑的 (pos, size, timestamp) 记下来（keydir.py:96-103）；
- 写 hint 时把本段的墓碑一起写进去，flags 用 `FLAG_TOMBSTONE`（segment.py:483-494）；加载时见到墓碑位就调 `keydir.delete()` 重放删除（segment.py:308-315）。

回归测试 `test_delete_survives_reopen_after_rotation` 专门钉住这个场景（tests/test_crash_recovery.py:46-55）。

### Bug 3 和 4：墓碑进了 hint 之后，重放顺序又塌了

把墓碑放进 hint 之后，fuzzer 立刻给了两个后坐力，这俩是一对：

**Compacted 段的 file id 比 active 段大。** 我的 compaction 给新段分配 `next_fid = self._active_file_id + 1`（engine.py:783）——压缩产物永远排在文件序列的末尾。于是重放顺序（按 file id 升序）不再等于时间顺序：一个存着**旧快照**的高 id 段，会排在存着**新写入**的低 id 段后面。按 file id 一路覆盖，最后活下来的是旧值。KeyDir.put 只好改成按记录时间戳仲裁：旧时间戳的迟到记录不许覆盖新条目（keydir.py:60-61）：

```python
if old.timestamp > entry.timestamp:
    return old                                     # keydir.py:60-61
```

墓碑同理：`keydir.delete` 带上墓碑自己的时间戳，比活条目旧的墓碑不许生效（keydir.py:88-90），但它的字节仍然计入死账。对应回归测试是 `test_compacted_replay_cannot_overwrite_newer_values` 和 `test_compacted_replay_cannot_resurrect_deleted_keys`（tests/test_crash_recovery.py:90-119）。

**同一毫秒内的时间戳并列。** 时间戳仲裁依赖"后写的时间戳更大"，但墙时钟毫秒精度做不到：delete 和紧跟着的 re-put 落在同一个毫秒里，两条记录时间戳一样大，重放顺序就悬了。修法是把"排序"和"过期"拆成两个时钟：排序用**逻辑写时钟** `_next_write_ts`，严格递增：

```python
self._last_write_ts = max(self._last_write_ts + 1, current_time_ms())   # engine.py:269
```

启动时用恢复出来的最大记录时间戳播种（engine.py:127，keydir.py:219-233），保证新会话的写入一定排在所有历史记录之后；而 TTL 的 `expire_at` 仍用墙钟（engine.py:354-355）——过期是和真实世界对齐的，不该被逻辑时钟带偏。

这四个 bug 的共同教训：**崩溃恢复的正确性不取决于任何一个组件单独对不对，取决于"重放所有来源"能否收敛到同一个状态**。hint、数据文件、compaction 产物，任何一个来源偷懒少带一点信息（少带墓碑、少带时间戳），收敛性就碎了。

## 五、TTL：惰性 + 主动，双淘汰

TTL 我做成了两层：

- **惰性淘汰**：读路径碰到就检查，`expire_at > 0` 且已过期就从 KeyDir 摘掉、计入死字节（keydir.py:40-46）。保证任何公开 API 都不会把过期 key 当活的返回。
- **主动清扫**：KeyDir 维护一个 `_ttl_keys` 集合（keydir.py:28），`purge_expired` 只遍历这个集合，O(T) 扫完全部带 TTL 的 key，不用全表扫（keydir.py:135-154）。`keys()`、`scan_prefix`、`scan_range` 这些入口都会先清一遍再返回（keydir.py:158-160），保证扫描结果里没有僵尸。

注意淘汰只是内存动作，磁盘上的记录还在，空间回收仍然是 compaction 的事——惰性淘汰 + compaction 是一个分层处理：读正确性靠前者，空间回收靠后者，互不催促。

fuzzer 的 ttl profile 顺带一提：它检查"5 秒 TTL 的 key 在 set 之后立刻 get 必须活着"。跑的时候 `findings.jsonl` 里留了一条 `TTL-EARLY-GONE`（seed 300153）：5 秒的 key "提前"过期了。我们没有把它当成第五个 bug 修——最合理的解释是 fuzz 进程在高负载机器上被调度器挂起了几秒，set 和 get 之间真实时间超过了 TTL；复跑 ttl profile 十几轮都没能复现。但它逼着我把 profile 的注释写明白："on a loaded machine even back-to-back statements can be descheduled for >100ms"（fuzz/model_fuzz.py:388-389，profile_ttl 的开头）——**测时间相关行为时，先想清楚你的断言依赖什么时钟假设**。

## 六、Compaction：两套实现和它们的原子性

compaction 只搬活数据：遍历 KeyDir 里的活条目，把 value 从旧段读出来、原样重写进新段（保留原时间戳和 expire_at），然后切流量、删旧段（engine.py:755-902）。两套实现：

- **单文件模式**：写到 `.compact.tmp`，sync 后 `os.replace` 原子替换主文件（engine.py:721）。崩溃在任何中间点，要么旧文件完整，要么新文件完整，不存在中间态。
- **多段模式**：先 rotate 把 active 段封存（这样所有候选段都不可变，期间新写入落到新 active 段），给压缩产物分配**更高的 file id**，给每个产物写 hint，然后才关旧段、删旧文件和旧 hint（engine.py:884-891），最后清掉对应的死字节计数和墓碑登记（engine.py:893-894）。

"值不值得搬"由死字节比例决定：`dead_bytes / total_disk >= compaction_threshold`（默认 0.5，config.py:51）才触发 auto_compact，且只在段轮转的时机顺带做（engine.py:287-288，engine.py:329-334）。死字节的账来自第二节说的那台"记账机"，另外还有个兜底：`_effective_dead_bytes` 会拿 `total_disk - live` 和记账值取 max（engine.py:324-327），防止记账漏账导致永远不触发。

诚实说一句代价：我的 compaction 全程持有引擎大锁（engine.py:664），期间写请求排队。生产级的 Bitcask（Riak 的实现）是把 merge 做成子进程、用文件系统级的替换 trick 避免阻塞。我选择先做对、再谈快——至少 fuzzer 证明了对。

## 七、写路径的三档 fsync 策略

 durability 是配置不是信仰，三档（segment.py:99-110）：

- `always`：每条 append 后 `flush + fsync`。最安全，最慢。
- `batch`：攒够 `sync_interval_records`（默认 128）条 fsync 一次。
- `os`（默认）：只 `flush` 到内核页缓存，进程崩溃不丢（数据已在内核），但断电可能丢。`close()` 和显式 `sync()` 时才真正 fsync。

`batch_write` 有个有意思的实现细节：批内每条记录都强制用 `os` 策略追加，整批写完才按用户配置补一次 fsync（engine.py:530-566）——不然一条批 100 个 put 在 `always` 档下就是 100 次 fsync，批处理就失去了意义。编码和校验全部先于任何磁盘写入完成，校验失败不会留下半截批（engine.py:488-525）。

## 面试官可能怎么追问

**Q1：KeyDir 必须全内存，10 亿个 key 怎么办？**
这是 Bitcask 的设计边界，不是 bug。Riak 论文的答案是把对象规模控制在内存能装下 key 的量级。真要上量，我能看到三条路：按 key 哈希分片成多个 Bitcask 实例，每个实例的 key 集独立装内存；只缓存热点 key 的索引、冷 key 走二级索引（牺牲 O(1) 点查）；或者承认场景变了换 LSM。我不会硬答"优化 KeyDir 的内存布局"——Python 对象换成紧凑数组能省 5-10 倍，救不了 100 倍的差距。

**Q2：崩溃时写了一半怎么办？CRC 只能检测，你怎么"恢复"？**
区分两种坏法。尾部撕裂（最后一条记录不完整或最后一条 CRC 错）符合 append-only 的物理模型，安全截断到上一条完整记录的边界；中间记录 CRC 错说明发生了模型之外的故障（位翻转、磁盘问题），我拒绝启动而不是静默截断——静默丢中间数据比拒绝服务更危险。这对应 segment.py:396-404 那个"只有尾记录才允许 CRC 截断"的分支。

**Q3：hint 文件自己坏了或者过期了怎么办？为什么不全靠 hint？**
hint 只是加速器，不是真相源，数据文件永远是真相源。我的信任条件是可自证的：hint 里所有记录覆盖的字节范围恰好等于数据文件大小才信（`_hint_covers_file`），每条 hint 记录自带 CRC。损坏（CRC 挂）或过期（大小不匹配）都退回全量扫描，慢但正确。设计上我学到的是：任何"摘要"必须携带足够信息自证它摘要的是哪一版本体。

**Q4：delete 只写墓碑，空间什么时候回收？compaction 期间还能写吗？**
墓碑落盘瞬间就把旧值的字节计入死字节统计，compaction 在死字节比例超阈值时触发（或段轮转时机顺带）。我的实现是先 rotate 把 active 段封存成只读候选，再全锁搬数据——所以 compaction 期间写会阻塞，但正确性简单可靠。工业实现（Riak 的 merge 进程）可以不阻塞写，代价是要处理"merge 期间新写入落在更新的段"的并发正确性。我承认这是我知道的、有意没做的简化。

**Q5：两次写发生在同一毫秒，重启后凭什么保证顺序？**
墙钟毫秒不够用，所以我根本不依赖墙钟排序。每条记录的时间戳来自逻辑写时钟：`max(last+1, now_ms)`，严格递增；启动时用已恢复记录的最大时间戳播种，新写入必然大于一切历史。仲裁规则是"旧时间戳的记录永不覆盖新条目"，对 put 和墓碑一致。这个问题是我 fuzzer 抓出来的（delete 后紧跟 re-put，重放顺序颠倒），不是我在设计时想到的——这也是我信模型测试多过信自己的原因。
