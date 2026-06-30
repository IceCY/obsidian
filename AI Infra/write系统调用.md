# write 系统调用：从调用到落盘

先记住一个核心点：

普通 `write()` 成功，不代表数据已经落盘。大多数情况下，它只是说明返回值对应的那部分数据已经被内核收下，放进了页缓存；真正写到磁盘，通常是后面的 writeback 或 `fsync()` 做的事。

大致路径：

```text
用户进程
  -> write()
  -> syscall 进入内核
  -> fd 找到 struct file
  -> VFS
  -> 具体文件系统
  -> page cache
  -> dirty page
  -> writeback
  -> block layer
  -> 驱动
  -> DMA
  -> SSD / HDD
```

## 用户态到内核态

应用代码里一般是：

```c
write(fd, buf, len);
```

这个 `write()` 通常先到 C 库。C 库做的事情不多，主要是按 ABI 把参数放进寄存器，然后执行系统调用指令。

以 x86-64 为例，大概是：

```text
rax = write 的系统调用号
rdi = fd
rsi = buf
rdx = len
syscall
```

执行 `syscall` 后，CPU 从用户态切到内核态。这里会发生上下文保存、权限级别切换、跳到内核 syscall 入口等动作。

这里的“保存上下文”要分两层看，不是 CPU 自动把所有寄存器都保存好。

```text
用户态执行 syscall
  -> 硬件保存最小返回现场
  -> 跳到内核 syscall entry
  -> Linux 入口代码切到内核栈
  -> 在内核栈上保存寄存器现场 pt_regs
  -> 执行具体系统调用
```

以 x86-64 为例，可以这么分：

| 谁做 | 保存 / 处理什么 | 放在哪里 | 作用 |
| --- | --- | --- | --- |
| CPU `syscall` 指令 | 用户态下一条指令地址 | `RCX` | 通常供 `sysret` 返回用户态，必要时内核也可能走 `iret` |
| CPU `syscall` 指令 | 用户态 `RFLAGS` | `R11` | 返回用户态时恢复标志位 |
| CPU `syscall` 指令 | syscall entry 地址、权限级别、部分 flags | MSR 配置决定入口 | 进入内核态执行 |
| Linux 入口代码 | 用户态栈指针、返回地址、段寄存器等返回现场 | 内核栈上的 `pt_regs` | 支持正确返回用户态 |
| Linux 入口代码 | 参数寄存器和通用寄存器，如 `rax`、`rdi`、`rsi`、`rdx`、`r10`、`r8`、`r9` 等 | 内核栈上的 `pt_regs` | 系统调用处理、信号、ptrace、调度 |
| Linux 入口代码 | 切换到当前线程的内核栈 | 内核栈 / per-cpu 状态 | 让内核代码在可信栈上运行 |

一个容易误解的点：x86-64 的 `syscall` 不会自动保存所有通用寄存器，`RSP` 也不是由 `syscall` 指令自动切到内核栈。这些主要是 Linux syscall 入口代码自己做的。

FPU / SIMD 寄存器通常不会在每次普通 syscall 入口都完整保存。它们由内核的 FPU 上下文机制在上下文切换或内核真正需要使用 FPU 时处理；否则每次系统调用都保存一大坨向量寄存器，成本会很高。

这一步本身就有成本。所以如果程序每次只写几个字节，会把大量时间浪费在系统调用和内核路径上。日志、网络、存储代码里经常强调 batch，本质上就是为了摊薄这些固定成本。

单看硬件指令层面的 `syscall` / `sysret` 往返，通常也比普通 `call` / `ret` 高一到两个数量级；具体可能是数百甚至上千个 cycle，取决于 CPU、内核配置和安全缓解措施。

## 内核先找到文件

进入内核后，内核不会直接相信用户传进来的东西。

它会先检查：

- `fd` 是否有效
- 当前文件是否允许写
- `buf` 指向的用户态地址范围是否大致可访问
- `len`、文件偏移、资源限制是否有问题

`fd` 只是进程文件描述符表里的一个下标。通过它可以找到内核里的 `struct file`，里面有打开方式、当前文件偏移、对应 inode 等信息。

这里几个对象要分清：

- `fd`：用户看到的整数
- `struct file`：一次 open 得到的打开文件对象
- `inode`：文件本身的元数据，比如大小、权限、块映射
- `dentry`：目录项缓存，路径名和 inode 的关系
- `address_space`：文件对应的页缓存管理结构

`struct file` 是一次打开产生的内核对象，不是文件本体。它不放在进程结构体内部，而是从内核的 file slab cache 这类全局内核对象池里分配；进程的 fd 表只保存指向它的指针。`dup()`、`fork()` 或 fd 传递可能让多个 fd 指向同一个 `struct file`，因此共享 `f_pos`；但两次独立 `open()` 会得到两个 `struct file`，它们可以指向同一个 inode 和 page cache，却各自维护文件偏移。

`write()` 用的是已经打开的 `fd`，所以正常情况下不需要再完整走一遍路径解析。路径解析主要在 `open()` 时发生。

## VFS 和具体文件系统

Linux 里应用统一调用 `write()`，但背后可能是 ext4、XFS、Btrfs、tmpfs、NFS、FUSE。VFS 就是中间那层统一接口。

VFS 负责把“对一个文件写入”这件事转成具体文件系统的操作。普通文件、socket、pipe、字符设备也都可以 `write()`，但后面的路径不一样。

这里主要看本地块设备上的普通文件。tmpfs、NFS、FUSE、socket、pipe、字符设备也能 `write()`，但后面的持久化和 I/O 路径不一样。

极简内核调用链可以先记成：

```text
ksys_write()
  -> vfs_write()
  -> file->f_op->write_iter()
  -> generic_file_write_iter() / iomap / buffered write
```

普通 buffered write 的核心动作是：

```text
copy_from_user(user buffer -> page cache)
```

也就是把用户态 `buf` 里的数据复制到内核的页缓存里。

关键对象关系大概是：

```text
task_struct
  -> files_struct
     -> fdtable[fd]
        -> struct file
           f_pos       当前文件偏移
           f_flags     O_APPEND / O_DIRECT 等打开标志
           f_mapping
             -> address_space
                i_pages   file page index -> page / folio
                host
                  -> inode
                     i_size    文件大小
                     i_mtime   修改时间
                     i_ctime   元数据变更时间
```

`page cache` 不是一个全局连续大 buffer，而是挂在文件 `address_space` 下面的一组缓存页。概念上按“文件偏移对应的页号”查找；实际实现里 folio 可能覆盖多个 page：

```text
page_index = file_offset / PAGE_SIZE
page_off   = file_offset % PAGE_SIZE
```

展开看，大概是：

| 步骤 | 读/写的信息 | 来源和保存位置 |
| --- | --- | --- |
| 1. 确定写入偏移 | 普通 `write()` 用 `struct file.f_pos`；`pwrite()` 用调用者传入的 offset；`O_APPEND` 会原子地以 inode 当前 `i_size` 作为写入起点 | `f_pos` 存在打开文件对象 `struct file` 里。`dup()` / `fork()` 共享同一个打开文件对象时也共享这个偏移；两次独立 `open()` 则各有自己的 `f_pos` |
| 2. 找到文件的 page cache | 通过 `struct file.f_mapping` 找到 `address_space`，再用 `page_index` 查 `address_space.i_pages` | `address_space` 通常挂在 inode 上，管理这个文件的缓存页；`i_pages` 现在通常是 XArray，key 是文件页号，value 是 page / folio |
| 3. 没有缓存页就创建 | 分配新的 page / folio，锁住后插入 `i_pages` | 页的元数据在内核的 `struct page` / `struct folio`，真实数据在物理内存页里。部分页覆盖时，文件系统可能需要先把旧内容读进来，避免没写到的字节被破坏 |
| 4. 从用户 buffer 复制 | `copy_from_user()` 把用户虚拟地址里的数据复制到 page cache 页的对应 offset | 源头是用户进程地址空间里的 `buf`；目标是内核管理的物理页。这里可能触发用户页缺页、权限检查、cache / TLB miss |
| 5. 更新文件状态 | 更新 `f_pos`、必要时更新 `inode.i_size`，并修改 `mtime` / `ctime` | `f_pos` 表示这次 `write()` 后下次顺序写的位置；`i_size`、时间戳存在 inode 里，是文件元数据 |
| 6. 标记 dirty | 把被改过的 page / folio 标记为 dirty，并把它挂到后续 writeback 能发现的结构上 | dirty 状态存在页/folio flags、`address_space` 标记和 inode / superblock 的 dirty/writeback 链路里；这表示“内存里的文件内容比磁盘新” |
| 7. 返回用户态 | 返回已经成功复制进 page cache 的字节数，或者错误码 | 返回值可能小于 `len`；后续写回错误也可能延迟到 `fsync()` / `close()` 等路径暴露 |

所以这一步结束后，后续是否能稳定保存，还要看 writeback、文件系统日志和设备 flush。

## 为什么要复制到页缓存

用户传进来的 `buf` 是用户态虚拟地址，内核不能长期拿着这个地址用。

原因有几个：

- 用户程序可能马上修改这块内存。
- 这段虚拟地址可能还没真正映射物理页。
- 地址可能非法，访问时会出错。
- 内核需要隔离用户态和内核态。
- 普通设备 DMA 也不应该随便访问任意用户态地址。

复制时还会牵涉 MMU、页表、TLB、page fault。比如用户 buffer 里有些页还没分配，`copy_from_user` 时就可能触发缺页。极端情况下，单次 `write()` 的延迟会因为 page fault 抖一下。

这层常见瓶颈：

- 小写太多，syscall 成本高
- 内存复制吃 CPU 和内存带宽
- page fault
- TLB / cache miss
- NUMA 远端内存访问

## 脏页和 writeback

被改过但还没写回磁盘的页叫 dirty page。`writeback` 就是内核把这些脏页从 page cache 刷到后端存储的机制。

普通 buffered write 的两段数据流：

```text
write() 路径：
用户 buffer
  -> copy_from_user()
  -> page cache
  -> 标记 dirty

writeback 路径：
dirty page / folio
  -> writeback worker
  -> 文件系统 writepages
  -> block mapping / extent mapping
  -> bio
  -> block layer
  -> 驱动
  -> 设备
```

触发 writeback 的常见来源：

| 触发源 | 作用 | 依赖的内核机制 |
| --- | --- | --- |
| 后台周期 | 周期性扫描，把变脏时间较久的页写回 | 主要靠 `delayed_work`：延迟阶段挂在内核 timer 上，到期后把 work 排进 writeback workqueue，由 `kworker` 执行 |
| dirty page 超过后台阈值 | 唤醒后台 writeback，避免脏页无限堆积 | 靠 dirty accounting 统计全局 / bdi / memcg 脏页；超过后台阈值后唤醒 flusher，也就是 queue writeback work |
| dirty page 超过更高阈值 | 写入进程会被 `balance_dirty_pages()` 限速，这就是 dirty throttling | 靠写入路径里的 dirty throttling 机制：`balance_dirty_pages()` 根据 dirty 限额、回写速度和设备能力让进程 sleep 或推动回写 |
| `fsync()` / `fdatasync()` / `sync()` | 用户显式要求把数据，必要时包括元数据，写到稳定存储 | 靠同步系统调用路径：VFS 从 file / superblock 找到 mapping 和 dirty inode，调用文件系统 writeback，并通过 wait queue 等待完成 |
| `O_SYNC` / `O_DSYNC` | 让 `write()` 返回前等待对应写回完成 | `O_SYNC` 接近 `fsync` 语义；`O_DSYNC` 接近 `fdatasync` 语义，只要求数据和读取这些数据所需的元数据 |
| 内存回收 | 回收 page cache 时，dirty page 需要先写回才方便释放 | 靠 vmscan / reclaim 机制：`kswapd` 或 direct reclaim 扫描 LRU，遇到 dirty folio 时触发 writepage/writeback 或等待已有回写 |
| 文件系统自身需求 | journal commit、unmount、freeze、delayed allocation 等路径可能推动写回 | 靠文件系统自己的事务和一致性机制：如 journal 线程、freeze / unmount 同步、delalloc 空间分配等主动发起回写 |

触发之后，writeback 先做的是把 dirty page 推进到块层，而不是同步等它们全部落盘：

```text
dirty inode 被挂到 writeback 队列
  -> 唤醒 writeback workqueue
  -> kworker 执行 writeback 工作
  -> 按 backing device / cgroup / superblock 选择 inode
  -> 扫描 inode 的 dirty folio
  -> 调用文件系统 writepages
  -> 生成 bio 并提交到块层
```

这里的 worker 不是应用线程，而是内核 workqueue 里的后台执行者。现代 Linux 主要用 `writeback` workqueue，实际在 `ps` 里常看到的是 `kworker/...`；执行写回时，worker 可能临时显示成类似 `flush-<设备>` 的名字。老资料里的 `pdflush` 是旧内核实现，不是现在的主路径。

writeback 相关的内核执行者可以这样分：

| 执行者 / 线程 | 负责什么 |
| --- | --- |
| `kworker/...` running `writeback` workqueue | 后台执行真正的 writeback 工作，扫描 dirty inode，调用文件系统回写接口 |
| `flush-<bdi>` | 不是另一套固定常驻线程，而是 writeback worker 执行时可能显示出来的名字；`bdi` 是 backing device info，代表后端设备写回域 |
| 用户进程自身 | 在 `fsync()`、`O_SYNC`、dirty throttling、直接回收等路径里，用户进程可能同步参与等待或推动写回 |
| 内存回收线程，如 `kswapd` | 内存压力下回收页缓存，遇到 dirty page 会触发或等待写回 |
| 文件系统 journal 线程，如 `jbd2/<dev>` | ext4 等日志文件系统的 journal 提交线程，负责元数据日志提交；它和 writeback 配合，但不等同于普通数据页 writeback |

这里的命名容易误导：真正的执行实体仍然是 `kworker`；`flush-<bdi>` 更像“这个 worker 正在做某个后端设备的回写”这种运行时身份标识。workqueue 机制本身见 [[workqueue机制]]。

更具体地说，主线内核里的 writeback workqueue 类似这样创建：

```c
bdi_wq = alloc_workqueue("writeback",
                         WQ_MEM_RECLAIM | WQ_UNBOUND | WQ_SYSFS,
                         0);
```

所以它底层走的是 **unbound normal worker pool**，常见执行线程名是 `kworker/u...`，而不是绑定某个 CPU 的 `kworker/0:...`。这个 pool 也不是每个 `bdi` 一个固定池；workqueue core 会按 unbound workqueue 的属性、NUMA / cpumask 等选择或复用对应的 unbound pool。`WQ_MEM_RECLAIM` 只是保证内存回收压力下有 rescuer 执行上下文，不表示绑定到特殊设备线程。

刚开始大量写文件时，速度可能很快，因为数据只是进了内存。等 dirty page 堆多了，内核会开始限制写入进程，让它等后端设备追上来。这就是 dirty throttling。

很多“写了一会儿突然卡住”的问题，其实不是 `write()` 本身变了，而是页缓存缓冲不住了。

### writeback worker 和 BIO 层怎么配合

这里要分清三类角色：

| 角色 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| writeback worker | 找到需要回写的 dirty inode / dirty folio，调用文件系统的 `writepages`，把脏页转换成块 I/O 请求 | 不直接操作硬件，不负责设备队列调度，也通常不等每个 BIO 真正写完 |
| 文件系统 | 把文件偏移映射到磁盘块，处理块分配、extent、journal / COW 元数据，构造 BIO | 不负责具体设备调度和驱动提交 |
| BIO / block layer | 表示一段块设备 I/O，把页缓存里的内存页和目标扇区组织起来，进入请求队列、合并、拆分、调度，再交给驱动 | 不知道“这是哪个文件的第几个字节”，只处理块设备上的扇区范围 |

换句话说，writeback worker 面向的是“文件和页缓存”，BIO / block layer 面向的是“块设备和扇区”。

一次典型异步回写可以拆成两条线：

```text
writeback worker 提交线：
dirty inode
  -> dirty folio / page
  -> lock folio
  -> 清 dirty 标记
  -> 设置 writeback 标记
  -> 文件系统映射文件偏移到磁盘块
  -> 把 folio 加进 bio
  -> submit_bio()
  -> 继续扫描和提交更多 I/O

BIO 完成线：
设备完成 I/O
  -> 驱动通知 block layer
  -> block layer 完成 request / bio
  -> 调用 bio->bi_end_io callback
  -> 文件系统 end_io 处理
  -> folio_end_writeback() / end_page_writeback()
  -> 清 folio writeback 标记
  -> 记录错误或唤醒等待者
```

`submit_bio()` 是两层之间最关键的交接点。它把 BIO 交给 block layer，但返回时通常只表示“已经提交给块层”，不表示数据已经写到设备，更不表示已经持久化。

这里的“提交”不是再丢给一个通用 workqueue，而是进入块层自己的 I/O 提交流水线。现代 Linux 的普通块设备主路径主要靠 **request queue + blk-mq，多队列块 I/O 机制**：

```text
文件系统构造 bio
  -> bio->bi_bdev 指向目标 block device
  -> bio->bi_iter.bi_sector 指定起始扇区
  -> bio 里挂 page / folio 对应的 bio_vec
  -> bio->bi_end_io 指定完成回调
  -> submit_bio(bio)
  -> block layer 检查队列限制、统计、拆分 / 合并
  -> 进入 request_queue
  -> blk-mq 把一个或多个 bio 组织成 request
  -> 按 CPU / NUMA 映射到软件队列 blk_mq_ctx
  -> 经过 plug / I/O scheduler / merge
  -> 派发到硬件队列 blk_mq_hw_ctx
  -> 调用驱动的 queue_rq 等提交入口
  -> 驱动把命令放进设备 submission queue / DMA ring
```

所以 `submit_bio()` 更像是“把已经描述好的块 I/O 放进块层队列体系”。它本身不负责睡眠等待设备完成；后续由 blk-mq、I/O scheduler、设备驱动和硬件队列继续推进。

其中几个机制容易混在一起：

| 机制 | 作用 |
| --- | --- |
| `struct bio` | 上层提交给块层的 I/O 描述：目标块设备、扇区、内存页、读写方向、完成回调 |
| `request_queue` | 每个块设备的块层队列入口，保存队列限制、调度器、blk-mq 配置等 |
| `blk-mq` | 现代块层多队列提交机制，把 BIO 合并 / 转换成 request，再分发到软件队列和硬件队列 |
| `blk_plug` | 批量提交优化。调用者短时间内要提交多个 BIO 时，块层可以先攒一下，方便合并，`blk_finish_plug()` 或任务睡眠时再冲出去 |
| I/O scheduler | 可选的请求调度层，负责排序、公平性和进一步合并；NVMe 等设备上也可能使用 `none` 这种很薄的调度 |
| 驱动 `queue_rq` | request 真正交给设备驱动的位置；比如 NVMe 驱动会把命令写入 NVMe submission queue |

文件系统 writeback 往往会在一段回写前后配合 `blk_start_plug()` / `blk_finish_plug()`，让一批相邻 BIO 有机会在块层合并成更少的 request。这是性能优化，不改变异步语义：`submit_bio()` 返回时，BIO 仍然只是进入了块层提交路径。

还有一类情况是 device mapper、md raid 这类 stacked block device。它们可能实现自己的 `submit_bio` 路径：先把上层 BIO 重映射、拆分或复制到下层设备，再继续 submit 到真正的底层块设备。因此从文件系统看仍然是 `submit_bio()`，中间可能经过一层或多层块设备映射。

所以后台 writeback worker 的典型行为是：

```text
submit_bio()
  -> 继续处理下一个 folio / inode
  -> 本轮 writeback 配额用完后退出
  -> 如果还有很多 dirty page，后面再被调度起来
```

它不是每提交一个 BIO 就同步等设备完成。否则后台回写会很难把设备队列打满，吞吐会很差。

提交 BIO 后，原来的 dirty page 不是被释放，也不是立刻从 page cache 里消失，而是换了状态：

```text
dirty folio
  -> 提交前清 dirty 标记
  -> 设置 writeback 标记
  -> I/O 完成后由 BIO completion callback 触发文件系统 end_io
  -> folio_end_writeback() / end_page_writeback() 清 writeback 标记
  -> 如果没有再次被写脏，就仍然留在 page cache，变成 clean folio
```

这里的 clean 不是一个被单独设置的 page flag，而是 dirty 和 writeback 都不存在时的结果：

```text
dirty    = PG_dirty
writeback = PG_writeback

clean folio ~= !PG_dirty && !PG_writeback
```

所以，“谁把它变成 clean”更准确地说是：

```text
writeback 提交路径先清 PG_dirty、设置 PG_writeback
BIO 完成路径通过 bio->bi_end_io 进入文件系统 end_io
文件系统 / iomap / buffer_head / mpage 等完成处理
  -> folio_end_writeback() / end_page_writeback()
  -> 清 PG_writeback
  -> 如果这期间没有再次 set dirty，该 folio 就表现为 clean
```

如果这个 folio 在 writeback 期间又被用户写了一次，它可能重新变 dirty。此时 BIO 完成路径仍然会清掉 `PG_writeback`，但 `PG_dirty` 还在，所以它不会变 clean，而是回到 dirty，后面还要再写一轮。

所以“移除”的含义要分层看：

| 对象 | 提交 BIO 后发生什么 |
| --- | --- |
| page / folio 本身 | 仍在 page cache 里；只是从 dirty 状态变成 writeback 状态，完成后再变成 clean |
| dirty 标记 / dirty 统计 | 提交写回前会清掉，并从 dirty 计数里扣除；否则同一页会被重复当成待回写页 |
| writeback 标记 / writeback 统计 | 提交 I/O 时设置，表示这页正在回写；BIO 完成回调触发文件系统完成路径，最终通过 `folio_end_writeback()` / `end_page_writeback()` 清掉 |
| inode 的 dirty/writeback 队列项 | worker 扫描时会从待处理队列取出；如果 inode 还有未提交的 dirty folio、还有 writeback 中的 folio，或本轮没处理完，可能重新挂到 dirty / io / more_io 等链表；如果都干净了，就从 writeback 相关队列里移除 |

因此，BIO 提交完成只表示“这批 folio 已经离开 dirty 待写集合，进入 writeback in-flight 集合”。真正能认为这批页已经干净，要等 BIO 完成回调清掉 folio 的 writeback 标记。

writeback worker 和 BIO 层之间主要靠这些状态和回调通信：

| 通信点 | 含义 |
| --- | --- |
| `struct bio` | 文件系统构造的块 I/O 描述，里面有目标块设备、扇区、参与 I/O 的 page / folio、完成回调等信息 |
| page / folio 的 dirty 标记 | 表示内存里的文件内容比磁盘新，需要 writeback |
| page / folio 的 writeback 标记 | 表示这个页已经提交或正在提交写回，不能简单当作普通 dirty page 再次提交 |
| `bio->bi_end_io` | BIO 完成后的 callback，设备 I/O 成功或失败都从这里往上通知 |
| `address_space` / mapping 的错误状态 | 如果写回失败，错误会记录在文件的 mapping 上，后续 `fsync()` 等调用可以把错误返回给用户 |
| wait queue / page wait bit | `fsync()`、truncate、reclaim 等路径可以等待某个 folio 的 writeback 状态清掉 |

BIO 完成后的 callback 机制大概是：

```text
设备完成 DMA / 写入命令
  -> 触发中断或轮询 completion
  -> block layer 标记 request 完成
  -> bio_endio(bio)
  -> bio->bi_end_io(bio)
  -> 文件系统检查 bio 状态
  -> 成功：结束相关 folio 的 writeback
  -> 失败：记录 writeback error，再结束 writeback
  -> folio_end_writeback() / end_page_writeback()
  -> 唤醒等待这个 folio / mapping 的任务
```

这里的 callback 通常不运行在原来的 writeback worker 上下文里。它可能运行在中断下半部、softirq、block completion 或相关内核线程上下文里。也就是说：

```text
提交 BIO 的线程：writeback worker / fsync 进程 / 其他内核路径
完成 BIO 的线程：block layer completion 路径
```

两者通过 BIO、folio 状态位和 completion callback 交接。

如果没有人在等，BIO 完成后主要就是清理状态、更新统计、释放 BIO。  
如果有人在等，比如 `fsync()`，等待者看到相关 folio 的 writeback 标记被清掉后，才会继续检查错误、提交必要元数据、下发 flush / FUA，然后返回用户态。

因此有一个重要结论：

```text
writeback worker 完成本轮工作
  != BIO 已经完成
  != 数据已经持久化
```

worker 负责“把脏页推进到块层”；BIO / block layer 负责“把块 I/O 推进到设备”；BIO callback 负责“设备完成后把结果通知回页缓存和文件系统”。

## 文件系统做什么

上面说的 `writepages` 是文件系统进入回写路径的核心入口。page cache 只知道“这个文件的第几个页变脏了”，真正写到设备前，文件系统要把文件偏移映射到块设备上的位置。

它要处理：

- 分配数据块
- 维护 extent / block mapping
- 更新 inode
- 更新文件大小
- 更新时间戳
- 写 journal 或 COW 元数据

覆盖写和追加写差别很大。

覆盖已有内容时，可能只需要改已有数据块。追加写时，往往要分配新块、更新文件大小、改 extent 树或空间位图，元数据成本更高。

文件系统还要考虑崩溃一致性。像 ext4、XFS 这类日志型文件系统，会把关键元数据写进 journal。这样系统崩溃后，文件系统结构还能恢复到一致状态。

但 journal 主要保证文件系统不坏，不等于保证应用刚写的数据一定还在。以 ext4 为例，`ordered`、`writeback`、`data=journal` 等 journal mode 对数据本身的保障也不一样。

Btrfs、OpenZFS / ZFS 这类 COW 文件系统思路不同：不原地覆盖旧块，而是写新块，再切换元数据指针。好处是快照和一致性更好，代价是写放大、元数据路径更重，随机写也可能更碎。

## 块层和驱动

文件系统准备好 BIO 后，会进入 block layer。到了这一层，文件语义基本已经消失，内核处理的是“对某个块设备的哪些扇区做 I/O”。

块层大概负责：

- 把相邻 I/O 合并
- 排队
- 调度
- 拆分过大的请求
- 处理 flush / barrier / FUA
- 控制队列深度
- 和 cgroup I/O 限制配合

对 HDD，调度的重点是减少寻道。对 SSD / NVMe，重点更多是并行度、队列深度和尾延迟。

再往下就是设备驱动。驱动把请求变成硬件命令，设置 DMA 描述符，然后提交给设备。

写文件时，数据从用户 buffer 到 page cache 这段通常是 CPU copy；从 page cache 到设备这段通常是 DMA。

完成后，设备通过中断或轮询告诉内核：这个 I/O 做完了。随后 block layer 走前面说的 BIO completion callback，把结果通知回文件系统和 page cache。

这里的瓶颈可能是：

- I/O 太小，合并不上
- 队列深度太低，设备吃不饱
- 队列太深，尾延迟变差
- flush 太频繁，设备无法重排
- 中断开销高
- IOMMU / 虚拟化路径增加延迟

## 到设备后也不一定已经持久化

设备收到写命令后，事情还没完全结束。

HDD 要等寻道、旋转、写扇区。随机写很慢，顺序写好很多。

SSD 没有机械寻道，但有 FTL、垃圾回收、磨损均衡、块擦除、SLC cache、写放大。随机小写多了，SSD 也会抖。

还有一个关键点：设备可能有 volatile write cache。设备报告“写完成”时，数据可能只是进了设备缓存，还没真正进非易失介质。

所以持久化语义要依赖：

- 文件系统正确下发 flush / barrier
- 块层正确传递 flush / FUA
- 设备正确实现这些命令
- SSD 最好有掉电保护

如果设备或虚拟化层谎报 flush 完成，`fsync()` 也救不了。

## `fsync()` 的位置

普通 `write()` 多数时候只到 page cache。`fsync(fd)` 才是把这个文件往持久化方向推的动作。

大概是：

```text
fsync(fd)
  -> 写回这个文件相关 dirty page
  -> 等待 I/O 完成
  -> 提交文件系统日志或必要元数据
  -> 下发设备 flush / FUA
  -> 返回
```

所以 `fsync()` 慢是正常的。它把之前异步攒着的工作都拿出来等。

`fdatasync()` 比 `fsync()` 稍弱一些，一般只要求数据和读取这些数据所需的元数据持久化。比如文件大小扩展相关的元数据必须同步，但纯时间戳这类不一定要。

还有一个细节：创建、rename 文件时，光 `fsync(file_fd)` 不一定够，目录项变化可能还要 `fsync(dir_fd)`。数据库、包管理器、配置文件原子替换会在意这个。

## 错误什么时候返回

buffered write 的错误不一定都在当次 `write()` 返回。比如 `EFAULT`、权限、部分空间不足可能当场暴露；但 `ENOSPC`、`EDQUOT`、`EIO` 也可能发生在后续 writeback 中，之后由 `fsync()`、`close()` 或后续相关写操作返回。

所以严肃写入通常要检查 `write()` 的返回值，处理 short write，并在需要持久化时检查 `fsync()`；如果依赖关闭时的结果，也不能完全忽略 `close()` 的返回值。

## 几个相关模式

`O_SYNC`：每次 `write()` 都等数据和相关文件完整性元数据同步完成再返回。语义强，慢。

`O_DSYNC`：类似 `O_SYNC`，但更偏数据，只要求数据和读取这些数据所需的元数据。

`O_DIRECT`：尽量绕过 page cache，让用户 buffer 和块设备直接做 I/O。它减少缓存和复制，但不等于自动落盘，通常仍然需要 `fsync()` 或同步语义。它也不保证完全零拷贝，可能有 bounce buffer、文件系统限制和 buffer / offset / length 对齐要求。

`mmap`：看起来是在写内存，其实是把文件页映射进地址空间。store 指令改页，页变 dirty，后面还是靠 writeback、`msync()` 或 `fsync()` 推到磁盘。

## 常见误解

`write()` 成功不等于落盘。

`fflush()` 不等于 `fsync()`。`fflush()` 多数只是把 C 标准库的用户态缓冲刷到内核。

`O_DIRECT` 不等于持久化。它主要解决绕过页缓存，不解决设备缓存和 flush。

`close()` 不等于 `fsync()`。但 `close()` 可能返回之前延迟记录的 writeback 错误，严肃程序不能完全忽略它。

SSD 随机写不是没有成本，只是比 HDD 好很多。FTL 和 GC 仍然会制造延迟抖动。

`fsync(file)` 不一定覆盖目录项变化，rename / create 场景要考虑父目录。

## 瓶颈怎么分层看

应用层：

- 写太小，syscall 太多
- 每条日志都 `fsync()`
- 多线程抢同一个文件偏移
- 序列化、压缩、加密本身太重

内存和内核层：

- `copy_from_user` 吃带宽
- page fault
- TLB / cache miss
- dirty page 太多导致 throttling

文件系统层：

- 频繁分配新块
- 元数据更新太多
- journal commit 太频繁
- 文件碎片
- COW 写放大

块层和设备层：

- I/O 太小
- 队列深度不合适
- flush / barrier 太频繁
- HDD 随机寻道
- SSD GC、写放大、SLC cache 用完
- 云盘或虚拟化后端抖动

## 一段能讲清楚的话

一次普通文件 `write()`，应用先通过系统调用从用户态进入内核。内核根据 `fd` 找到打开文件对象，经 VFS 进入具体文件系统。对 buffered I/O，内核把用户 buffer 里的数据复制到文件的 page cache，更新文件偏移、大小和时间戳，把页标记为 dirty，然后 `write()` 就可以返回。

后面由后台 writeback 或 `fsync()` 把 dirty page 交给文件系统。文件系统完成块映射、块分配、journal 或 COW 元数据处理，再把 I/O 提交到块层。块层合并、排队、调度请求，驱动设置 DMA，把数据交给 SSD 或 HDD。真正要保证持久化，还要等文件系统日志、必要元数据以及设备 flush / FUA 完成。

所以分析写入性能时，要分清卡在哪里：是 syscall 太碎、内存复制太重、dirty page 被限速、文件系统元数据太多，还是块设备和 SSD / HDD 本身跟不上。
