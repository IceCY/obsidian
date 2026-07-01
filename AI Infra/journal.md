# Linux 文件系统日志机制

这里的 journal 不是应用层日志，而是文件系统为了处理崩溃一致性做的写前日志。

核心问题是：一次文件系统操作通常要改多个磁盘位置。比如创建一个文件，至少可能涉及：

- 父目录的数据块：增加一个目录项 `name -> inode`
- inode 位图：标记某个 inode 已分配
- inode 表：写入新 inode 的模式、时间戳、大小、块映射等
- block 位图 / extent tree：如果分配了数据块，也要更新块分配信息
- superblock / group descriptor：更新空闲计数

如果系统在这些写入中间掉电，磁盘上可能出现“目录项指向一个未初始化 inode”、“inode 占用但位图说空闲”、“块已经分给文件但空闲位图仍然可分配”等状态。日志系统的目标不是保证每个应用刚写的数据都不丢，而是保证文件系统元数据不会停在半更新状态。

可以先记住一句：

```text
journal 保证的是文件系统更新的原子性和可恢复性；
普通 write() 的数据持久性仍然取决于 writeback、fsync、journal mode 和设备 flush。
```

它正好接在 [[write系统调用]] 后面。`write()` 把数据放进 page cache 后，后续 writeback 和文件系统日志共同决定哪些东西以什么顺序进入块设备。

## 为什么需要日志

最朴素的方案是每次修改都直接原地写：

```text
修改 inode 位图
修改 inode 表
修改目录块
修改 group descriptor
```

这个方案的问题是磁盘 I/O 不是天然事务。哪怕单个扇区/块写入本身有一定原子性，文件系统的一次逻辑操作也往往跨多个块。崩溃可能发生在任意两个写之间。

老式做法是挂载时跑 `fsck`，全盘扫描修复不一致。这个办法能用，但代价很高：

- 大盘全量扫描很慢
- 修复逻辑复杂，可能只能把损坏内容放进 `lost+found`
- 不能很好支持快速重启

journaling 的思路是：真正改主文件系统区域之前，先把“这次要改什么”顺序写到一块日志区域里。日志区域写完并带有 commit 记录后，这个事务才算成立。崩溃后只需要扫描 journal，把已经提交但还没完全落到主区域的事务 replay 一遍即可。

## 基本模型

把一次文件系统更新抽象成事务：

```text
transaction T:
  修改 block A
  修改 block B
  修改 block C
```

journal 写入顺序大概是：

```text
1. 把 A/B/C 的新内容或变更描述写到 journal
2. 确认 journal 内容已经到稳定存储
3. 写 commit block，表示事务 T 完整提交
4. 之后再把 A/B/C 写回它们在主文件系统里的最终位置
5. 等主区域写完，这部分 journal 空间可以回收
```

崩溃恢复时：

- 看到完整 commit：说明事务已经提交，可以 replay
- 没看到 commit，或者 checksum 不对：说明事务没完整提交，丢弃
- replay 是幂等的：重复把同一批元数据块写到最终位置，结果仍然一致

这就是 write-ahead logging。关键在于 commit 记录是事务边界。

## ext4 和 JBD2

Linux 里 ext4 使用 JBD2 作为日志层。JBD2 是 Journaling Block Device 的第二版，不只 ext4 能用，OCFS2 也使用它。ext4 把“我要修改哪些元数据块”交给 JBD2，JBD2 负责事务组织、journal 空间管理、commit、checkpoint 和 recovery。

对象关系可以粗略看成：

```text
ext4
  -> 决定哪些 inode / bitmap / extent / dirent 需要修改
  -> 估算本次操作需要多少 journal credits
  -> jbd2_journal_start()
       得到 handle
  -> 修改受日志保护的 buffer
  -> jbd2_journal_stop()

JBD2
  -> 把多个 handle 合并进 running transaction
  -> commit thread 异步提交事务
  -> 管理 journal ring、revoke、checkpoint、recovery
```

几个核心概念：

| 概念 | 作用 |
| --- | --- |
| `journal_t` | 一个 journal 实例，记录日志设备、大小、head/tail、commit 线程、当前事务等 |
| `transaction_t` | 一个日志事务，包含一批元数据 buffer 的修改 |
| `handle_t` | 文件系统一次操作在事务里的句柄，带有 credits，表示预计要修改多少日志块 |
| credits | 日志空间预算。ext4 开始修改前要告诉 JBD2 大概要碰多少块，避免事务写到一半没空间 |
| commit thread | 通常能看到类似 `jbd2/<dev>-8` 的内核线程，负责把事务提交到 journal |
| checkpoint | 已提交事务里的元数据写回主文件系统位置后，journal 空间才能释放 |

JBD2 的事务是批处理的。多个进程的多个文件系统操作可以进入同一个 running transaction。这样可以把许多小元数据更新合成一次 journal commit，吞吐更好；代价是单个操作可能要等事务关闭和提交，尤其是 `fsync()` 这种同步语义。

## 一次 ext4 元数据更新

以 `mkdir /mnt/fs/d` 为例，忽略权限和缓存细节，元数据路径大概是：

```text
sys_mkdirat()
  -> VFS 路径解析
  -> ext4_mkdir()
  -> ext4_journal_start()
  -> 分配 inode
  -> 初始化目录 inode
  -> 修改父目录数据块，增加 dentry
  -> 修改 inode bitmap / inode table / group descriptor
  -> ext4_mark_inode_dirty()
  -> ext4_journal_stop()
```

中间每次要改受保护的元数据 buffer 前，ext4 会告诉 JBD2：“我要写这个 buffer，请把旧状态纳入事务管理”。修改后再把 buffer 标为属于当前事务。commit 时，JBD2 把这些 buffer 的新内容写进 journal。

更抽象一点：

```text
进程上下文:
  handle = jbd2_journal_start(journal, credits)
  get_write_access(metadata_buffer)
  修改 metadata_buffer
  dirty_metadata(metadata_buffer)
  jbd2_journal_stop(handle)

JBD2 commit 线程:
  关闭 running transaction
  写 descriptor block
  写 journal data blocks / metadata blocks
  写 revoke blocks
  下发 flush / barrier
  写 commit block
  必要时再 flush
  事务进入 checkpoint 阶段
```

这里的 “data blocks” 容易误会。在 JBD2 语境里，journal data block 指的是“写进 journal 的块内容”。在 ext4 默认模式下，它们通常是文件系统元数据块，不一定是普通文件的数据块。

## journal 的磁盘格式

ext4 内部 journal 通常是隐藏 inode，典型是 inode 8；也可以放在外部 journal 设备上。journal 本身像一个循环日志区域。

一个传统 JBD2 full commit 大概长这样：

```text
journal superblock

transaction N:
  descriptor block
    tag: 最终要写到主文件系统的 block A
    tag: 最终要写到主文件系统的 block B
  block A 的新内容
  block B 的新内容
  revoke block 可选
  commit block

transaction N+1:
  ...
```

各部分含义：

| journal 块 | 作用 |
| --- | --- |
| superblock | 记录 journal 大小、起始位置、序列号、特性、checksum 等 |
| descriptor block | 描述后面跟着哪些块，以及这些块最终要写回主文件系统的哪个 block number |
| data block | 被日志保护的块内容本身 |
| revoke block | 记录某些旧 journal 记录不应再 replay |
| commit block | 标记事务完整提交。恢复时没有 commit 的事务会被忽略 |

revoke 的典型场景是：某个块以前作为元数据被 journal 记录过，后来这个块被释放并重新分配成普通文件数据块。如果崩溃恢复时把旧的元数据 journal 记录 replay 回去，就会覆盖新的文件数据，造成损坏。因此需要 revoke 记录告诉恢复逻辑跳过旧记录。

commit block 是最重要的事务边界。它表示前面的 descriptor、数据块、revoke 信息已经组成一个完整事务。恢复时根据事务序列号、commit block、checksum 判断哪些事务有效。

## checkpoint 不是 commit

commit 和 checkpoint 要分清：

```text
commit:
  事务已经安全写入 journal
  崩溃后可以靠 replay 恢复

checkpoint:
  事务影响的元数据块已经写回主文件系统位置
  这段 journal 空间可以被复用
```

所以事务 commit 后，主文件系统区域里的 inode 表、位图、目录块不一定已经全部更新完。但这时已经安全：只要 journal commit block 在，崩溃后 replay 可以补齐。

这也解释了为什么 journal 空间满时，新的文件系统操作可能会被迫等待：不是因为 commit 没做，而是旧事务还没 checkpoint，journal ring 没有足够可用空间。

## ext4 的三种 data mode

ext4 的 journal 行为由 data mode 影响。这里的 data 指普通文件数据，不是元数据。

| 模式 | 日志内容 | 崩溃后的典型语义 | 性能 |
| --- | --- | --- | --- |
| `data=ordered` | 只 journal 元数据，但提交相关元数据前，会先把对应脏数据块写出去 | 默认模式。避免新元数据指向未写出的垃圾数据，但不保证最近写入一定存在 | 折中 |
| `data=writeback` | 只 journal 元数据，不强制数据块先于相关元数据写出 | 文件系统结构一致，但崩溃后新文件里可能出现旧数据或未预期内容 | 通常更快 |
| `data=journal` | 文件数据和元数据都先写 journal，再写最终位置 | 数据和元数据都被 journal 保护，语义最强 | 写放大最大，通常最慢 |

默认 `data=ordered` 很容易被说成“数据也有日志”，但严格说不是。它 journal 的仍主要是元数据，只是给数据块和元数据 commit 建立了顺序约束：

```text
普通文件数据块先写到最终位置
相关元数据再 commit 到 journal
```

这个顺序约束很关键。假设扩展文件大小并分配了新块：

```text
write("hello") 写入 page cache
writeback 分配磁盘块 D
inode extent 指向 D
inode size 变大
```

如果 inode size 和 extent 先 journal commit，而 D 的数据没写出去，崩溃后 replay 会让 inode 指向 D，但 D 里可能是旧内容。`data=ordered` 会在提交这类元数据前，推动相关数据块先落盘。

不过它仍不等于 `fsync()`。如果应用只是 `write()` 后立刻掉电，这次写入可能还没进入需要提交的事务，也可能数据页还在 page cache 里。

## fsync 和 journal

`fsync(fd)` 的语义是：把这个文件的数据和必要元数据推进到稳定存储，使得成功返回后，崩溃恢复还能看到这次写入。

对 ext4 这类日志文件系统，`fsync()` 大概会做几类事：

```text
1. 写出这个 inode 相关的 dirty data pages
2. 确保分配块、extent、i_size、mtime 等必要元数据进入 journal
3. 强制相关 journal transaction commit
4. 等待必要 I/O 完成
5. 下发设备 flush / FUA，处理 volatile write cache
```

这里最容易踩坑的是目录项。创建或 rename 文件后，如果只 `fsync(file)`，文件内容可能稳定了，但父目录里的“名字 -> inode”关系不一定稳定。很多可靠写文件模式会这么做：

```text
write temp file
fsync(temp file)
rename(temp, final)
fsync(parent directory)
```

原因是 `rename()` 的原子性主要是命名空间元数据事务的原子性；要保证崩溃后目录项也持久，需要同步父目录。

## rename 为什么常被认为原子

以 `rename("a.tmp", "a")` 为例，文件系统需要修改目录项：

```text
删除 / 替换旧的 a
把 a.tmp 的目录项改成 a
更新 inode link count / ctime 等
```

这些元数据更新会放进同一个 journal transaction。崩溃恢复后，事务要么没有 commit，被丢弃；要么 commit 完整，被 replay。因此用户看到的结果通常是：

- 旧 `a` 还在
- 新 `a` 出现

而不是目录结构处于一半状态。

但这不自动保证新 `a` 的文件数据已经落盘。可靠更新配置文件时，仍然需要先 `fsync(tmp)`，再 `rename()`，再 `fsync(dir)`。

## crash recovery

挂载 ext4 时，如果发现 journal 不是 clean 状态，就会进入 recovery。JBD2 recovery 大致分三步：

```text
1. 扫描 journal，找到日志尾部和有效事务范围
2. 收集 revoke 记录
3. replay 所有已提交、未被 revoke 的事务块
```

replay 的动作不是“重新执行 mkdir/write/rename 的代码”，而是更底层地把 journal 里记录的块内容写回它们的最终 block number。也就是说，journal 记录的是块级结果，而不是系统调用级操作。

恢复完成后，文件系统元数据回到某个事务边界上的一致状态。最近没提交的事务会丢，已经提交但未 checkpoint 的事务会补写。

## 和 page cache / writeback 的关系

普通 buffered write 的路径是：

```text
write()
  -> copy_from_user 到 page cache
  -> folio 标 dirty
  -> 返回用户态
```

journal 主要参与后面的文件系统元数据路径：

```text
dirty data page
  -> writeback
  -> ext4 分配块 / 更新 extent / 更新 inode size
  -> 元数据修改进入 JBD2 transaction
  -> journal commit
  -> checkpoint
```

元数据和数据页不是一回事：

| 类型 | 例子 | 是否默认 journal |
| --- | --- | --- |
| 文件数据 | 用户写入的 `"hello"` | ext4 默认 `data=ordered` 下不 journal |
| 文件元数据 | inode size、mtime、extent tree | journal |
| 空间管理元数据 | block bitmap、inode bitmap、group descriptor | journal |
| 命名空间元数据 | 目录项、link count | journal |

delayed allocation 会让事情更绕一点。ext4 可能在 `write()` 时先不分配真实磁盘块，只记录 page cache 里有脏数据；等 writeback 或 `fsync()` 时再分配块并更新 extent。这样性能和布局更好，但也意味着块分配相关元数据可能更晚才进入 journal。

## 设备 flush 和 barrier

journal 的正确性依赖写入顺序：

```text
journal 数据块必须先于 commit block 稳定
commit 成功后，恢复逻辑才能信任这个事务
```

但现代磁盘和 SSD 可能有 volatile write cache，普通写完成不一定意味着掉电不丢，也不一定严格按提交顺序落到介质。因此文件系统和 block layer 要配合 flush / FUA / barrier 语义：

- 在写 commit block 前，确保事务内容已经稳定
- 在需要强持久性时，确保 commit block 自身也稳定
- `fsync()` 成功返回前，要把相关缓存刷新到稳定存储

如果设备谎报 flush 完成，或者存储栈错误地丢掉 barrier，journal 的崩溃一致性也会被破坏。

## fast commit

新一些的 ext4 支持 fast commit。传统 full commit 会把完整元数据块写进 journal；fast commit 记录的是某些操作的最小 delta，例如：

- 给 inode 增加一个 extent range
- 删除某段 logical offset
- 创建 / 删除 / link 某个目录项
- 更新 inode 相关状态

当 fast commit 区域满了、遇到不适合 fast commit 的操作，或者 JBD2 commit timer 到期时，ext4 会退回传统 full commit。一次 full commit 会使之前的 fast commit 区域失效，后面再重新积累 fast commit。

fast commit 的重点是降低提交延迟，而不是改变 journal 的基本语义。恢复时仍然要能把记录 replay 成一致的文件系统状态。

## 和其他文件系统的对比

不是所有 Linux 文件系统都按 ext4/JBD2 这个模型来。

| 文件系统 | 一致性思路 |
| --- | --- |
| ext4 | JBD2 日志，默认 metadata journaling + ordered data |
| XFS | 自己的元数据日志系统，不使用 JBD2；默认也主要保护元数据 |
| Btrfs | COW 思路，写新块并通过元数据树切换根；也有 log tree 优化 fsync |
| F2FS | 面向 flash 的 log-structured/checkpoint 思路 |
| tmpfs | 内存文件系统，不涉及掉电后磁盘恢复 |

所以“Linux 文件系统日志系统”没有一个完全统一的 VFS journal 层。VFS 提供 inode、dentry、file、superblock 等抽象；真正的 crash consistency 策略在具体文件系统内部。

## 典型崩溃例子

### 创建文件但不 fsync

```c
fd = open("a", O_CREAT | O_WRONLY, 0644);
write(fd, "hello", 5);
close(fd);
// 立刻掉电
```

可能结果：

- `a` 不存在
- `a` 存在但长度为 0
- `a` 存在且内容是 `hello`

文件系统应该保证的是：目录、inode、位图等结构不损坏。它不保证这次没有 `fsync()` 的应用数据一定存在。

### 创建文件并 fsync 文件

```c
fd = open("a", O_CREAT | O_WRONLY, 0644);
write(fd, "hello", 5);
fsync(fd);
close(fd);
```

`hello` 和文件必要元数据更有保障。但如果这个文件名是新创建的，严格的可移植可靠模式还要 `fsync()` 父目录，因为目录项本身属于父目录元数据。

### 原子替换文件

```c
fd = open("a.tmp", O_CREAT | O_WRONLY | O_TRUNC, 0644);
write(fd, new_config, len);
fsync(fd);
close(fd);
rename("a.tmp", "a");
dirfd = open(".", O_RDONLY | O_DIRECTORY);
fsync(dirfd);
close(dirfd);
```

这里利用的是：

- 文件数据通过 `fsync(fd)` 稳定
- `rename()` 的目录元数据更新通过 journal 保持原子
- 父目录 `fsync(dirfd)` 保证命名变化稳定

## 性能代价

journal 带来的主要成本：

- 写放大：元数据先写 journal，再写主区域
- flush 成本：commit 边界需要持久化顺序，设备 flush 可能很贵
- 事务等待：`fsync()` 可能要等待当前 transaction commit
- journal 空间压力：checkpoint 跟不上时，新事务会等待
- 小同步写放大：每条记录都 `fsync()` 会导致频繁 commit

优化方向通常是 batch：

- 多条应用日志合并后再 `fsync()`
- 使用 group commit
- 避免每次小文件创建都同步目录
- 根据风险选择 `data=ordered` / `data=journal`
- 对数据库这类系统，让数据库 WAL 和文件系统 journal 的同步策略配合好

## 小结

文件系统 journal 是内核处理崩溃一致性的机制。它把一次复杂的元数据更新包装成事务：先顺序写 journal，写完整后提交 commit block，再慢慢 checkpoint 到主文件系统区域。崩溃恢复时，只 replay 已经完整提交的事务，丢弃未提交事务，从而避免文件系统结构停在半更新状态。

以 ext4 为例，JBD2 管理事务、commit、revoke、checkpoint 和 recovery；ext4 决定哪些元数据需要被 journal 保护。默认 `data=ordered` 只 journal 元数据，但要求相关数据块先于元数据提交写出。它保证文件系统结构一致，不等于保证普通 `write()` 返回后的数据一定持久。真正的持久化边界通常还是 `fsync()`，并且要考虑目录 `fsync()`、journal commit、checkpoint、block layer 和设备 flush。

参考：

- Linux Kernel Docs: [ext4 Journal (jbd2)](https://docs.kernel.org/filesystems/ext4/journal.html)
- Linux Kernel Docs: [The Linux Journalling API](https://docs.kernel.org/filesystems/journalling.html)
- Linux Kernel Docs: [ext4 data mode](https://docs.kernel.org/admin-guide/ext4.html#data-mode)
