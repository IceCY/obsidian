# Linux block layer

这篇主要讲现代 Linux blk-mq 路径下，`bio` 怎么变成 `request`，再怎么交给块设备驱动。文件名虽然叫 `bio.md`，但内容范围是整个 block layer 主路径。

版本口径：

```text
以现代主线内核的 blk-mq 心智模型为准
具体结构体字段、helper 名字和回调细节要以目标 kernel tree 为准
```

读法建议：

```text
只想理解 bio:
  看到 bvec_iter / I/O operation 和 flags 即可

想理解 bio -> request:
  重点看 bdev、merge、request、blk-mq 映射和派发

想读或写驱动:
  重点看 queue_rq()、blk_mq_ops、完成路径、virtio-blk / NVMe 例子
```

先记住一个核心点：

block layer 是 Linux 内核里连接“上层 I/O 语义”和“块设备驱动”的中间层。

上层看到的是：

```text
文件系统 / page cache / direct I/O / swap
  -> 我要读写某个 block device 的一段逻辑 sector
  -> 数据在这些内存页里
```

设备驱动看到的是：

```text
struct request
  -> 操作类型
  -> 起始 sector / 长度
  -> scatter-gather 数据段
  -> tag / queue / timeout / completion
```

block layer 负责把中间这段路打通：

```text
上层构造 bio
  -> submit_bio()
  -> block layer 检查、拆分、合并、调度、限流、统计
  -> blk-mq 把 bio 组织成 request
  -> 调用块设备驱动的 queue_rq()
  -> 驱动把 request 翻译成 virtio / NVMe / SCSI / SATA 等设备命令
  -> 设备完成
  -> 驱动通知 block layer
  -> block layer 完成 request / bio
  -> bio->bi_end_io()
```

最重要的三层对象是：

```text
bio      = 上层 I/O 描述单元
request  = block layer 调度和驱动提交单元
command  = 设备协议命令，比如 NVMe command 或 virtio descriptor
```



## 层次关系

完整路径可以画成：

```text
文件系统 / swap / direct I/O / page cache writeback
  -> struct bio
  -> submit_bio()
  -> block layer
       -> queue limits 检查
       -> bio split / clone / remap
       -> blk plug
       -> merge
       -> I/O scheduler
       -> request_queue
       -> blk-mq software queue: blk_mq_ctx
       -> blk-mq hardware context: blk_mq_hw_ctx
       -> struct request
  -> block device driver
       -> blk_mq_ops.queue_rq()
       -> 驱动构造设备命令
       -> DMA / virtqueue / submission queue / doorbell
  -> hardware / virtual backend
```

所以，“BIO 层”和 “block layer” 不应该理解成两个并列层。

更准确地说：

```text
struct bio 是 block layer 暴露给上层的 I/O 描述对象
submit_bio() 是上层进入 block layer 的入口
blk-mq / request_queue / scheduler 是 block layer 内部到驱动之间的提交机制
```

平时说“BIO 层”，通常是在强调 block layer 靠近文件系统 / page cache 的这一侧。

## block layer 负责什么

block layer 的职责可以分成几类：


| 职责             | 说明                                                                            |
| -------------- | ----------------------------------------------------------------------------- |
| I/O 表示         | 用 `struct bio` 表示上层 I/O，用 `struct request` 表示调度和驱动提交单元                        |
| 设备抽象           | 用 `struct gendisk`、`struct block_device`、`struct request_queue` 抽象磁盘、分区和队列    |
| 队列限制           | 根据 logical block size、max sectors、max segments、DMA alignment 等限制拆分 I/O        |
| 合并和调度          | 把相邻 I/O merge 成更大的 request，必要时通过 I/O scheduler 排序和公平分配                        |
| 多队列提交          | 用 blk-mq 把 per-CPU software queue 映射到 hardware queue，提高多核和高速设备并行度             |
| 完成路径           | 从驱动 completion 回到 request，再回到 bio，再回到文件系统 / page cache / direct I/O 等上层       |
| flush / FUA    | 处理 volatile write cache、持久化顺序和 cache flush                                    |
| 资源控制           | 队列深度、tag、request allocation、cgroup I/O 控制、统计、throttling                       |
| 特殊能力           | discard、write zeroes、zoned block device、integrity、inline encryption、polling 等 |
| stacked device | 支持 dm、md、loop、crypt 等块设备层层映射和转发 I/O                                           |


它不负责：


| 不负责什么                           | 谁负责                      |
| ------------------------------- | ------------------------ |
| 文件 offset 到磁盘块的映射               | 文件系统                     |
| page cache dirty / writeback 状态 | mm + 文件系统                |
| 构造具体设备协议命令                      | 设备驱动                     |
| SSD FTL、磨损均衡、垃圾回收               | 设备固件                     |
| 虚拟机后端真正写到哪里                     | host / hypervisor / 后端存储 |




## 关键对象总览

先看全局对象关系：

```text
gendisk
  -> 表示一个磁盘设备，比如 nvme0n1、vda、sda
  -> 拥有 request_queue
  -> 暴露 /sys/block/<disk>

block_device
  -> 表示被打开的块设备或分区，比如 /dev/nvme0n1p1
  -> 指向 gendisk 和 partition 信息

request_queue
  -> 这个磁盘的块层队列
  -> 保存 queue limits、scheduler、blk-mq tag_set、统计、sysfs 属性

bio
  -> 上层提交的 I/O 描述
  -> 目标 bdev、sector、operation、bio_vec、end_io

request
  -> block layer 给驱动的调度单元
  -> 可以包含一个或多个 bio

blk_mq_ctx
  -> per-CPU software staging queue

blk_mq_hw_ctx
  -> block layer 的硬件队列上下文
  -> 通常映射到设备驱动的某个 submission queue / DMA ring / virtqueue

blk_mq_tag_set
  -> 驱动声明的 blk-mq 队列能力和回调集合
  -> 保存 blk_mq_ops、queue depth、hardware queue 数量、driver private data 大小等
```



## `struct bio`

`bio` 是上层提交给 block layer 的 I/O 描述。

它表达的是：

```text
对哪个 block device
从当前 sector 开始
做什么操作
数据在内存的哪些 page / segment 里
完成后调用哪个 callback
```

一个普通 read/write bio 的设备侧通常是一段连续逻辑 sector：

```text
bio->bi_bdev              目标 block device
bio->bi_iter.bi_sector    当前 iterator 的起始 sector
bio->bi_iter.bi_size      当前剩余 I/O 字节数
```

内存侧不要求连续：

```text
bio
  -> bio_vec[0]: page A + offset + len
  -> bio_vec[1]: page K + offset + len
  -> bio_vec[2]: page B + offset + len
```

所以可以这样记：

```text
bio 的设备侧：一段逻辑连续的 sector range
bio 的内存侧：一组不一定连续的 bio_vec
```

这里的“连续”是对 `bi_bdev` 的逻辑地址空间来说的，不保证真实物理介质连续。dm、md、virtio、SSD FTL 都可能继续映射。

主线内核里常见字段可以这么理解。`struct bio` 不是稳定 ABI，不同内核版本和配置项会有调整；读代码时优先看目标内核源码和 helper，不要把下面这张表当成固定布局。


| 字段                 | 类型                               | 含义                                                      |
| ------------------ | -------------------------------- | ------------------------------------------------------- |
| `bi_next`          | `struct bio *`                   | request queue 或 bio 链表里的下一个 bio                         |
| `bi_bdev`          | `struct block_device *`          | 目标块设备                                                   |
| `bi_opf`           | `blk_opf_t`                      | operation + flags，低位是 `REQ_OP_*`，高位是 `REQ_*` flags      |
| `bi_flags`         | `unsigned short`                 | BIO 内部状态标志，比如 cloned、chained、remapped 等                 |
| `bi_ioprio`        | `unsigned short`                 | I/O priority                                            |
| `bi_write_hint`    | `enum rw_hint`                   | 写入 lifetime hint，给设备或块层做写入归类提示                          |
| `bi_write_stream`  | `u8`                             | 写 stream 标识，和写入归类 / stream 相关                           |
| `bi_status`        | `blk_status_t`                   | 完成状态，成功或具体块层错误                                          |
| `bi_io_vec`        | `struct bio_vec *`               | `bio_vec` 数组，描述数据分布在哪些内存页里                              |
| `bi_iter`          | `struct bvec_iter`               | 当前剩余 I/O 的迭代状态                                          |
| `bi_end_io`        | `bio_end_io_t *`                 | I/O 完成回调                                                |
| `bi_private`       | `void *`                         | 上层自定义上下文，completion callback 常用它找回文件系统状态                |
| `bi_blkg`          | `struct blkcg_gq *`              | cgroup I/O 归属信息，依赖 `CONFIG_BLK_CGROUP`                  |
| `bi_crypt_context` | `struct bio_crypt_ctx *`         | inline encryption 上下文，依赖 `CONFIG_BLK_INLINE_ENCRYPTION` |
| `bi_integrity`     | `struct bio_integrity_payload *` | data integrity payload，依赖 `CONFIG_BLK_DEV_INTEGRITY`    |
| `bi_vcnt`          | `unsigned short`                 | bio 中有效 `bio_vec` 数量。驱动代码通常不应依赖它迭代 bio                  |
| `bi_max_vecs`      | `unsigned short`                 | 这个 bio 最多能容纳多少个 `bio_vec`                               |
| `__bi_cnt`         | `atomic_t`                       | bio 引用计数 / pin count                                    |
| `bi_pool`          | `struct bio_set *`               | bio 来源的 bio set，用于释放和复用                                 |




## `struct bio_vec`

`bio_vec` 描述 bio 里的一个内存片段。

概念上是：

```c
struct bio_vec {
        struct page *bv_page;
        unsigned int bv_len;
        unsigned int bv_offset;
};
```

也就是：

```text
从 bv_page 的 bv_offset 位置开始
长度 bv_len 字节
这段内存参与本次 I/O
```

`bv_offset` 是相对 `bv_page` 起点的字节偏移，所以结构上通常应该落在这个 page 内：

```text
0 <= bv_offset < PAGE_SIZE
```

`bv_offset` 不天然要求按 512B 或 4K 对齐。真正的对齐要分开看：

```text
块设备地址对齐：看 bio 当前 iterator 的 bi_sector、剩余长度、logical block size
内存 DMA 对齐：看 page_to_phys(bv_page) + bv_offset 是否满足队列 dma_alignment 等限制
```

direct I/O 更容易看到非 0 的 `bv_offset`，因为用户 buffer 地址可能从某个 page 中间开始。

现代内核里 biovec 在 submit 后应该视为不可变；如果 I/O 部分完成，推进的是 iterator，而不是直接改原始 `bio_vec`。

## `struct bvec_iter`

`bvec_iter` 记录 bio 当前处理到哪里。

一个容易混的点：`bi_iter.bi_sector` 不是“永远不变的原始起始 sector”。它是 iterator 的一部分，表示当前剩余 I/O 的起始 sector。

bio 刚构造出来时：

```text
bi_iter.bi_sector = 原始起始 sector
bi_iter.bi_size   = 总长度
```

如果 block layer 或驱动对 bio 做了 advance：

```text
原始 bio: sector 1000, size 128K
完成前 64K 后:
  bi_iter.bi_sector = 1000 + 64K / 512
  bi_iter.bi_size   = 64K
```

所以更严谨的说法是：

```text
bi_sector = 当前 iterator 位置对应的 sector
```

常见字段：


| 字段 / 概念        | 含义                                     |
| -------------- | -------------------------------------- |
| `bi_sector`    | 当前剩余 I/O 的起始设备扇区，单位通常是 512-byte sector |
| `bi_size`      | 剩余 I/O 字节数                             |
| `bi_idx`       | 当前处理到第几个 `bio_vec`                     |
| `bi_bvec_done` | 当前 `bio_vec` 内已经处理了多少                  |


驱动和 block layer 迭代 bio / request 时应该使用 helper，比如：

```text
bio_for_each_segment()
bio_for_each_bvec()
rq_for_each_segment()
rq_for_each_bvec()
```

不要手写逻辑去直接依赖 `bi_vcnt`、`bi_idx` 之类字段，尤其是在 cloned / split / partially completed bio 上，很容易错。

## I/O operation 和 flags

`bi_opf` 包含 operation 和 flags。

常见 operation：


| 操作                    | 含义                                |
| --------------------- | --------------------------------- |
| `REQ_OP_READ`         | 从设备读到内存                           |
| `REQ_OP_WRITE`        | 从内存写到设备                           |
| `REQ_OP_FLUSH`        | 要求设备把 volatile write cache 刷到持久介质 |
| `REQ_OP_DISCARD`      | 通知设备某些块不再需要，SSD 上类似 TRIM          |
| `REQ_OP_WRITE_ZEROES` | 要求设备把一段范围写成 0，可能由设备高效完成           |
| `REQ_OP_ZONE_APPEND`  | zoned device 上追加写                 |
| passthrough 操作        | 向驱动或设备传递协议相关命令                    |


常见 flags：


| flag           | 含义                                      |
| -------------- | --------------------------------------- |
| `REQ_SYNC`     | 同步倾向的 I/O，调度上可能更重视延迟                    |
| `REQ_FUA`      | Force Unit Access，要求这次写直接达到非易失介质或等价持久状态 |
| `REQ_PREFLUSH` | 写前先 flush，常用于持久化顺序约束                    |
| `REQ_META`     | 元数据 I/O                                 |
| `REQ_RAHEAD`   | readahead 读                             |
| `REQ_IDLE`     | 这批 I/O 后可能空闲，给调度器提示                     |


普通 `REQ_OP_WRITE` 完成不一定等于数据掉电不丢。设备有 volatile write cache 时，普通写完成可能只是写进设备缓存。真正的持久化语义要靠文件系统、block layer 和设备一起正确处理 flush / FUA。

## `struct request`

`request` 是 block layer 给驱动的提交单元。

它和 bio 的关系是：

```text
一个 bio
  -> 可能变成一个 request

多个相邻 bio
  -> 可能 merge 成一个 request

一个太大的 bio
  -> 可能 split 成多个 bio / request
```

request 面向驱动，里面包含：


| 信息                   | 说明                                          |
| -------------------- | ------------------------------------------- |
| `q` / request_queue  | 所属块层队列                                      |
| operation / flags    | read、write、flush、discard、FUA 等              |
| sector / bytes       | 当前请求的设备逻辑范围                                 |
| bio 链表               | request 中包含的一个或多个 bio                       |
| tag                  | blk-mq 分配的 tag，驱动可用它对应硬件 command id         |
| timeout              | 请求超时时间                                      |
| scheduler / mq state | 调度和派发状态                                     |
| driver private data  | `tag_set->cmd_size` 让每个 request 后面附带驱动私有命令区 |


驱动通常不直接手拆 bio，而是用 block layer helper：

```text
blk_rq_pos(rq)        当前 request 起始 sector
blk_rq_bytes(rq)      当前 request 剩余字节数
req_op(rq)            operation
rq_data_dir(rq)       read / write 方向
rq_for_each_segment() 遍历 request 的单页 segment
rq_for_each_bvec()    遍历 request 的 bvec
blk_rq_map_sg()       把 request 映射成 scatterlist
blk_mq_rq_to_pdu()    找到 request 后面的驱动私有数据区
```

可以把 request 理解成：

```text
block layer 已经帮驱动整理过的一批 bio
```



## `struct request_queue`

`request_queue` 是块设备的队列核心对象。

它保存：


| 内容            | 说明                                                     |
| ------------- | ------------------------------------------------------ |
| queue limits  | 最大请求大小、最大 segment 数、logical block size、DMA alignment 等 |
| blk-mq 信息     | tag set、ctx / hctx 映射、queue depth、hardware queues      |
| I/O scheduler | `none`、`mq-deadline`、`bfq`、`kyber` 等                   |
| flush queue   | flush / FUA 相关状态                                       |
| accounting    | iostats、latency、in-flight、cgroup 归属等                   |
| sysfs 属性      | `/sys/block/<disk>/queue/*`                            |
| queue state   | frozen、quiesced、dying、stopped、congested 等状态            |


一个磁盘通常有一个 `request_queue`。分区共享底层磁盘的队列。

对 stacked block device，比如 dm、md、loop，队列可能主要用于承载映射层限制和 sysfs 信息，真正的 I/O 会被 remap 到下层设备。

## queue limits

queue limits 是 block layer 做拆分、合并和对齐检查的依据。

常见限制：


| 限制                           | 含义                                            |
| ---------------------------- | --------------------------------------------- |
| `logical_block_size`         | 设备可寻址的最小逻辑块大小                                 |
| `physical_block_size`        | 设备物理写入原子单位或偏好的物理块大小                           |
| `io_min` / `minimum_io_size` | 不造成明显性能惩罚的偏好最小 I/O 大小                         |
| `io_opt` / `optimal_io_size` | 设备偏好的持续 I/O 大小，比如 RAID stripe width           |
| `max_sectors`                | block layer 允许的最大请求大小                         |
| `max_hw_sectors`             | 硬件支持的最大单次传输大小                                 |
| `max_segments`               | request 中允许的最大 scatter-gather segment 数       |
| `max_segment_size`           | 单个 segment 最大长度                               |
| `seg_boundary_mask`          | segment 不能跨越的地址边界                             |
| `dma_alignment`              | DMA buffer 地址对齐要求                             |
| discard limits               | discard 粒度、最大 discard 大小、最大 discard segment 数 |
| zoned limits                 | zone size、max open zones、max active zones 等   |


这些限制影响：

```text
bio 能不能直接提交
bio 是否要 split
两个 bio / request 能不能 merge
direct I/O 是否满足对齐
驱动最终收到的 scatterlist 是否符合硬件限制
```

比如：

```text
文件系统提交 1MB bio
  -> 设备 max_sectors 只允许 256KB
  -> block layer 拆成 4 个 request
```

或者：

```text
request 中 segment 太多
  -> 超过 max_segments
  -> block layer split
```

sysfs 里可以看到很多队列属性：

```text
/sys/block/<disk>/queue/logical_block_size
/sys/block/<disk>/queue/physical_block_size
/sys/block/<disk>/queue/max_sectors_kb
/sys/block/<disk>/queue/max_segments
/sys/block/<disk>/queue/max_segment_size
/sys/block/<disk>/queue/write_cache
/sys/block/<disk>/queue/scheduler
/sys/block/<disk>/queue/rotational
/sys/block/<disk>/queue/nomerges
```



## blk-mq

blk-mq 是现代 Linux 的多队列 block I/O 提交机制。

它解决的问题是：老的单队列块层在多核和高速 SSD / NVMe 上锁竞争太重，无法把设备并行度喂满。

blk-mq 把队列分成两级：

```text
software staging queue: blk_mq_ctx
  -> 通常按 CPU 或 CPU group 分布
  -> 减少提交路径锁竞争

hardware dispatch queue: blk_mq_hw_ctx
  -> block layer 面向驱动的硬件队列上下文
  -> 通常映射到设备的 submission queue、virtqueue、DMA ring 或 host queue
```

大致路径：

```text
submit_bio()
  -> blk_mq_submit_bio()
  -> 尝试 merge
  -> 分配 request 和 tag
  -> 放入当前 CPU 的 blk_mq_ctx
  -> 映射到 blk_mq_hw_ctx
  -> scheduler 或 dispatch list
  -> 调用 driver queue_rq()
```

如果路径很短，block layer 可能直接把 request 派发给驱动；如果需要 merge、scheduler 或设备暂时忙，就会在软件队列或 dispatch list 里停一下。

## bio 如何变成 request

一个容易误解的点是：

```text
bio 不是简单地 memcpy 成 request
```

更准确地说：

```text
bio 是上层提交的一段 I/O 描述
request 是 block layer 为了调度和驱动提交而建立的容器
request 里会挂一个或多个 bio
```

进入 request-based blk-mq 队列后，普通 bio 的主线大致是：

```text
submit_bio()
  -> submit_bio_noacct()
  -> bio->bi_bdev 找到 request_queue
  -> queue limits / split / remap / accounting
  -> blk_mq_submit_bio()
       -> 先尝试 merge 到已有 request
       -> merge 失败则分配新的 request
       -> 把 bio 挂到 request 的 bio 链上
       -> 插入 plug / scheduler / dispatch 队列
       -> 最终调用驱动 queue_rq()
```



## bdev 怎么找到 request_queue

`bio` 里不会直接保存 `request_queue *`，它保存的是目标块设备：

```text
bio->bi_bdev
  -> struct block_device
```

这个 `bi_bdev` 通常不是 submit 路径临时通过 `/dev/xxx` 字符串查出来的。

更早的时候，上层已经拿到了目标块设备：

```text
mount / open / filesystem setup
  -> 打开某个 block device
  -> 得到 struct block_device
  -> 文件系统、swap、direct I/O 路径保存这个 bdev
```

真正构造 bio 时，上层会把目标 bdev 填进去：

```text
构造 bio
  -> bio_set_dev(bio, bdev)
  -> bio->bi_bdev = bdev
  -> bio->bi_iter.bi_sector = 这个 bdev 地址空间里的 sector
```

所以 submit 路径已经站在一个具体 `block_device` 上了，它要做的是沿着内核对象指针找到队列，而不是重新做设备名解析。

现代内核里 `struct block_device` 自己就有队列指针：

```text
struct block_device
  -> bd_disk     所属 gendisk
  -> bd_queue    所属 request_queue
```

所以 submit 路径里常见的查找非常直接：

```c
struct block_device *bdev = bio->bi_bdev;
struct request_queue *q = bdev_get_queue(bdev);
```

而 `bdev_get_queue(bdev)` 概念上就是：

```c
return bdev->bd_queue;
```

`bd_queue` 又是在 block device 创建时从 disk 上带下来的：

```text
gendisk
  -> queue

block_device 初始化
  -> bdev->bd_disk = disk
  -> bdev->bd_queue = disk->queue
```

因此普通整盘设备可以这样看：

```text
bio->bi_bdev
  -> /dev/nvme0n1 对应的 block_device
  -> bdev->bd_queue
  -> nvme0n1 这个 disk 的 request_queue
```

普通分区设备也差不多：

```text
bio->bi_bdev
  -> /dev/nvme0n1p1 对应的 block_device
  -> bdev->bd_queue
  -> 仍然是 nvme0n1 整个 disk 的 request_queue
```

也就是说，分区不是每个都有一套独立硬件队列。分区 `block_device` 和整盘 `block_device` 通常共享同一个底层 `request_queue`。

区别在于 sector 要做一次分区偏移换算。

比如文件系统挂在 `/dev/nvme0n1p1` 上，它构造 bio 时：

```text
bio->bi_bdev = nvme0n1p1 的 block_device
bio->bi_iter.bi_sector = 分区内 sector
```

如果 p1 从整盘 sector 2048 开始：

```text
bio 原始位置:
  bdev   = nvme0n1p1
  sector = 1000

partition remap 后:
  sector = 1000 + 2048 = 3048
```

这样后面的 queue limits、merge、request 起始 sector、驱动看到的 `blk_rq_pos(rq)`，都基于整盘地址空间。

所以“根据 bdev 找 request_queue”其实分成两件事：

```text
找队列:
  bio->bi_bdev->bd_queue

修正地址:
  如果 bio->bi_bdev 是分区
  -> bio->bi_iter.bi_sector += 分区在整盘里的起始 sector
```

实际代码通常通过 helper 取分区起始位置，不建议读代码时死记某个字段名；不同内核版本的 `struct block_device` 字段会变。

还有一类情况要分开：stacked / bio-based block device。

如果目标 `bdev` 的 disk 自己实现了 `submit_bio`，block layer 不会马上把 bio 变成 request，而是先交给这个设备的 `disk->fops->submit_bio(bio)`：

```text
bio 指向 /dev/dm-0
  -> dm 的 submit_bio()
  -> dm 查映射表
  -> split / clone / remap
  -> 下发新的 bio 到真正的底层 bdev
```

等 bio 最终落到 request-based blk-mq 设备时，才进入：

```text
bdev_get_queue()
  -> blk_mq_submit_bio()
  -> bio merge / request allocation / dispatch
```

也就是说，bio 到 request 的转换通常有两种结果：

```text
bio 被 merge 到已有 request
  -> 没有新 request
  -> 原 request 的长度、segment 信息、bio 链表被扩展

bio 不能 merge
  -> 分配一个新 request
  -> 用这个 bio 初始化 request 的 sector、bytes、op、flags、segment 信息
```



## 先尝试 merge

block layer 会先看这个 bio 能不能并入已有 request。

这里的“已有 request”不是指系统里任意一个 request，也不是已经交给硬件的所有 request。

它指的是：

```text
同一个 request_queue / blk-mq 队列上下文里
还停留在 block layer 可合并位置上的 request
```

常见位置有几类：


| 位置                        | 说明                                                        |
| ------------------------- | --------------------------------------------------------- |
| 当前线程的 plug list           | `blk_start_plug()` 后提交的一批 request 先攒在当前任务本地，最容易和后续 bio 合并 |
| scheduler 队列              | mq-deadline、bfq、kyber 等调度器持有的 request，还没派发给驱动             |
| software queue / ctx 相关结构 | blk-mq per-CPU 提交侧暂存的 request，具体形态随 scheduler 和内核版本变化     |
| dispatch list             | 已经准备派发但暂时没能交给驱动的 request，某些情况下仍可能参与合并                     |


一旦 request 已经真正进入驱动 / 硬件 in-flight 状态，通常就不能再把新的 bio 合并进去了：

```text
已经调用 queue_rq()
  -> 驱动可能已经构造设备命令
  -> DMA / virtqueue / NVMe SQ 可能已经提交
  -> request 的设备侧命令边界基本固定
  -> 后来的 bio 不能再塞进去
```

所以 merge 发生的窗口主要是在：

```text
bio 提交进 block layer
  -> request 仍在 plug / scheduler / software queue / dispatch 等块层内部位置
  -> 还没变成驱动已经消费的设备命令
```

常见的 merge 方向有两类：

```text
back merge:
  已有 request: sector 1000, len 4K
  新 bio:       sector 1008, len 4K
  -> bio 接到 request 后面

front merge:
  新 bio:       sector 1000, len 4K
  已有 request: sector 1008, len 4K
  -> bio 接到 request 前面
```

能不能 merge 不只看 sector 是否相邻，还要看一组条件是否兼容：


| 条件                 | 说明                                                     |
| ------------------ | ------------------------------------------------------ |
| 目标设备               | 必须是同一个底层队列语义下的 I/O                                     |
| operation          | read 不能和 write 合并，discard / flush / write zeroes 也各有规则 |
| flags              | FUA、sync、metadata、priority 等语义不能冲突                     |
| queue limits       | 合并后不能超过 max sectors、max segments、segment boundary 等限制  |
| integrity / crypto | integrity payload、inline encryption 上下文要兼容             |
| cgroup / ioprio    | I/O 归属和优先级通常要能放在同一个 request 里                          |


merge 成功后，bio 只是成为某个 request 的一部分：

```text
request
  -> bio A
  -> bio B
  -> bio C
```

驱动后来看到的是一个 request，但 completion 时 block layer 仍然会按 request 里的 bio 链逐个完成原来的 bio。

## merge 失败才创建新 request

如果找不到可合并的 request，或者队列设置禁止 merge，就会为这个 bio 分配新的 request。

这一步会做几件事：

```text
分配 struct request
  -> 选择当前 CPU 对应的 blk_mq_ctx
  -> 映射到一个 blk_mq_hw_ctx
  -> 分配 tag 或准备稍后分配 tag
  -> 初始化 request 的 op、flags、起始 sector、长度
  -> 把第一个 bio 挂到 request->bio / request->biotail 链上
```

可以粗略理解成：

```text
request 的地址范围来自第一个 bio
request 的数据段来自 bio_vec 迭代结果
request 的 bio 链保存原始上层 completion 边界
```

request 自己有面向驱动和调度的状态，比如 tag、timeout、mq context、scheduler 信息；bio 仍然保留上层关心的完成回调和 private data。

## blk_mq_ctx 到 blk_mq_hw_ctx 的映射

这里的“映射”不是磁盘 sector 映射，也不是虚拟地址到物理地址映射。

它指的是：

```text
当前提交 I/O 的 CPU
  -> 使用哪个 blk_mq_ctx
  -> 这个 ctx 对应哪个 blk_mq_hw_ctx
  -> request 最后从哪个 hctx 派发给驱动
```

可以把它理解成 blk-mq 的 CPU 到硬件提交队列的路由表。

`blk_mq_ctx` 是提交侧的 per-CPU 软件上下文：

```text
CPU 0 -> ctx0
CPU 1 -> ctx1
CPU 2 -> ctx2
CPU 3 -> ctx3
```

但硬件队列数量通常不等于 CPU 数量。比如 4 个 CPU，设备只有 2 个硬件提交队列，映射可能是：

```text
CPU 0 / ctx0 -> hctx0
CPU 1 / ctx1 -> hctx0
CPU 2 / ctx2 -> hctx1
CPU 3 / ctx3 -> hctx1
```

如果设备只有一个硬件队列，那就更简单：

```text
CPU 0 / ctx0 -> hctx0
CPU 1 / ctx1 -> hctx0
CPU 2 / ctx2 -> hctx0
CPU 3 / ctx3 -> hctx0
```

如果是 NVMe 这类多队列设备，`hctx` 往往会对应某个 NVMe submission queue / completion queue pair；virtio-blk 里可能对应某个 virtqueue。

这个映射不是 submit_bio() 临时拍脑袋算出来的。它在设备初始化 blk-mq 队列时就建好了。

可以分成两层看：

```text
第一层：tag_set 级别
  建 CPU -> 硬件队列编号 的 queue map

第二层：request_queue 级别
  根据 queue map 创建 hctx
  再把 per-CPU ctx 挂到对应 hctx 上
```



## 软件队列到硬件队列的映射什么时候完成

时间点在块设备注册 / request_queue 初始化阶段，而不是每次 I/O 提交时。

简化路径是：

```text
驱动准备 blk_mq_tag_set
  -> nr_hw_queues = 设备想暴露几个硬件队列
  -> queue_depth = 每个硬件队列能挂多少 request
  -> ops = blk_mq_ops
  -> 可选 map_queues() 回调

blk_mq_alloc_tag_set()
  -> 分配 tag_set 里的 tags / queue map 等全局 blk-mq 资源
  -> blk_mq_update_queue_map()
  -> 如果驱动实现了 map_queues()
       -> 调用驱动的 map_queues()
       -> 让驱动指定 CPU 应该落到哪些硬件队列
  -> 否则 block layer 用默认策略建立 CPU -> queue index 映射

blk_mq_alloc_disk() / blk_mq_init_queue()
  -> 创建 request_queue
  -> 根据 tag_set->nr_hw_queues 创建 q->queue_hw_ctx[] 数组
  -> 为每个 CPU 准备 blk_mq_ctx
  -> blk_mq_map_swqueue()
  -> 根据 queue map 把 ctx 归到对应 hctx
  -> 调用驱动 init_hctx() 初始化每个 hctx
```

所以，驱动初始化后大概形成这样的结构：

```text
request_queue q
  -> q->queue_ctx       per-CPU blk_mq_ctx
  -> q->queue_hw_ctx    blk_mq_hw_ctx 数组

tag_set
  -> map[HCTX_TYPE_DEFAULT]
       -> mq_map[cpu] = hctx_index
```

概念上可以看成有一张表：

```text
queue_map[cpu] = hctx_index
```

更贴近数据结构一点，可以这样记：

```text
struct blk_mq_tag_set
  -> map[type].mq_map[cpu] = hctx_index
  -> nr_hw_queues
  -> ops

struct request_queue
  -> queue_ctx[cpu]          当前 queue 的 per-CPU blk_mq_ctx
  -> queue_hw_ctx[index]     当前 queue 的 blk_mq_hw_ctx

struct blk_mq_ctx
  -> hctxs[type]             运行时快速找到目标 hctx

struct blk_mq_hw_ctx
  -> queue
  -> ctxs[]                  归到这个 hctx 的 ctx 集合
  -> cpumask
  -> driver_data             驱动挂自己的硬件队列上下文
```

`tag_set->map[type].mq_map[]` 是“CPU 到硬件队列编号”的原始路由表；`q->queue_hw_ctx[]` 是“编号到 hctx 对象”的数组；`ctx->hctxs[type]` 是 `blk_mq_map_swqueue()` 建好后的运行时缓存。这样提交路径不用每次重新组织 ctx / hctx 关系。

比如 8 个 CPU、4 个硬件队列，可能是：

```text
mq_map[0] = 0
mq_map[1] = 0
mq_map[2] = 1
mq_map[3] = 1
mq_map[4] = 2
mq_map[5] = 2
mq_map[6] = 3
mq_map[7] = 3
```

然后 request_queue 初始化时会把 ctx 组织到 hctx 下面：

```text
hctx0
  -> ctx of CPU0
  -> ctx of CPU1

hctx1
  -> ctx of CPU2
  -> ctx of CPU3

hctx2
  -> ctx of CPU4
  -> ctx of CPU5

hctx3
  -> ctx of CPU6
  -> ctx of CPU7
```

驱动的 `map_queues()` 为什么要参与？

因为只有驱动最清楚硬件队列和 CPU / MSI-X 中断 / NUMA 的关系。

比如 NVMe 设备可能希望：

```text
CPU 绑定到离自己近的 submission queue / completion queue
某些 CPU share 一个 queue
poll I/O 使用单独 poll queue
read queue 和 default queue 分开
```

如果驱动不提供特殊策略，block layer 会用默认映射，尽量把 CPU 均匀分布到可用硬件队列上。

提交 I/O 时：

```text
当前 CPU = smp_processor_id()
ctx = 当前 CPU 对应的 blk_mq_ctx
hctx_index = queue_map[当前 CPU]
hctx = q->queue_hw_ctx[hctx_index]
```

实际代码会通过 blk-mq helper 完成这件事，而不是裸写上面这些伪代码。读代码时常见的核心意思是：

```text
用当前 CPU 和 request_queue 的 queue map
找到这个 request 应该归属的 blk_mq_hw_ctx
```

源码里可以重点认这几个名字：


| 名字                                               | 作用                                             |
| ------------------------------------------------ | ---------------------------------------------- |
| `blk_mq_update_queue_map()`                      | 更新 `tag_set->map[]`，也就是 CPU 到硬件队列编号的映射         |
| `blk_mq_map_queues()`                            | block layer 默认映射策略，驱动没有 `map_queues()` 时使用     |
| `blk_mq_ops.map_queues()`                        | 驱动自定义映射，比如按 MSI-X、NUMA、设备队列拓扑分配 CPU            |
| `blk_mq_map_swqueue()`                           | 把每个 CPU 的 `blk_mq_ctx` 挂到对应 `blk_mq_hw_ctx` 下面 |
| `blk_mq_map_queue()` / `blk_mq_map_queue_type()` | submit 路径按 I/O 类型和当前 CPU 找到目标 `hctx`           |


`blk_mq_map_swqueue()` 做的事情可以近似理解成：

```text
for_each_possible_cpu(cpu):
  ctx = per_cpu q->queue_ctx[cpu]

  for each hctx map type:
    hctx_index = set->map[type].mq_map[cpu]
    hctx = q->queue_hw_ctx[hctx_index]

    ctx->hctxs[type] = hctx
    hctx->ctxs[] 添加 ctx
    hctx->cpumask 添加 cpu
```

这样 submit 路径就不用重新建立关系，只需要查：

```text
当前 CPU 的 ctx
  -> ctx->hctxs[当前 I/O 类型]
  -> hctx
```

为什么要这么做？

```text
减少多个 CPU 抢同一把队列锁
让提交路径尽量保持 CPU local
把 I/O 分散到设备的多个硬件提交队列
让 tag 分配、dispatch、completion affinity 更容易按队列组织
```

还有一个细节：现代 blk-mq 可能有不同类型的 hctx map。

比如：


| map 类型            | 用途                  |
| ----------------- | ------------------- |
| default queue map | 普通 read / write I/O |
| read queue map    | 某些设备可为 read 单独准备队列  |
| poll queue map    | polled I/O 使用的队列    |


所以更完整地说，映射不只是：

```text
CPU -> hctx
```

而是：

```text
I/O 类型 + 当前 CPU -> hctx
```

例如一个 writeback worker 在 CPU 2 上提交普通 write bio：

```text
writeback worker 当前跑在 CPU 2
  -> 选择 CPU 2 的 blk_mq_ctx
  -> 查普通 I/O queue map
  -> CPU 2 映射到 hctx1
  -> 新 request 归属 hctx1
  -> 之后从 hctx1 调用 driver queue_rq()
```

把查找流程写成伪代码，大概是：

```text
type = 根据 request 类型选择 HCTX_TYPE_DEFAULT / READ / POLL
cpu  = 当前提交 CPU

ctx = q->queue_ctx[cpu]

如果 swqueue 映射已经缓存：
  hctx = ctx->hctxs[type]

概念上等价于：
  hctx_index = q->tag_set->map[type].mq_map[cpu]
  hctx = q->queue_hw_ctx[hctx_index]

request 归属 hctx
blk-mq 后续用这个 hctx 调用 driver queue_rq(hctx, bd)
```

注意，block layer 在这一层只找到 `blk_mq_hw_ctx`。至于这个 `hctx` 在驱动里对应 NVMe SQ、virtio virtqueue 还是别的 DMA ring，是驱动在 `init_hctx()` 之类的初始化回调里绑定的。

## request 插入哪里

新 request 建好之后，不一定马上调用驱动。

它可能进入几个位置：


| 位置            | 什么时候出现                                             |
| ------------- | -------------------------------------------------- |
| plug list     | 当前线程正在 plug，短时间内攒一批 I/O，等 `blk_finish_plug()` 再冲出去 |
| scheduler     | 队列启用了 mq-deadline、bfq、kyber 等调度器                   |
| dispatch list | request 已经准备派发，但硬件队列暂时忙或 tag / 资源不够                |
| 直接 dispatch   | `none` scheduler、队列空闲、资源足够时可能直接进驱动                 |


最后真正到驱动时，接口是：

```text
blk_mq_ops.queue_rq(hctx, bd)
  -> bd->rq 是 block layer 准备好的 request
```



## ctx 和 hctx 运行时怎么交互

前面讲的是静态映射：

```text
CPU / blk_mq_ctx -> blk_mq_hw_ctx
```

运行时还要看 request 怎么从 per-CPU 提交侧流到 per-hctx 派发侧。

可以先用两阶段模型理解：

```text
阶段 1：提交侧，偏 per-CPU / ctx
  -> 当前 CPU 提交 bio
  -> 找到当前 CPU 的 blk_mq_ctx
  -> 计算这个 I/O 的目标 hctx，用来决定后面该运行哪条硬件队列
  -> 尝试 plug merge / scheduler merge / request merge
  -> 新 request 可能暂存在 plug、scheduler，或 ctx 的 software queue
  -> 这里只是确定归属，不等于已经把 request 放进 hctx

阶段 2：派发侧，偏 per-hctx / hctx
  -> run 这个 hctx
  -> 从它管理的多个 ctx 或 scheduler 里取 request
  -> 放到 hctx dispatch list 或直接准备派发
  -> 分配 / 确认 driver tag
  -> 调用 driver queue_rq(hctx, bd)
```

关键数据结构可以这么记：

```text
struct blk_mq_ctx
  -> lock
  -> rq_lists[type]       这个 CPU 提交侧暂存的 request 链表
  -> hctxs[type]          这个 ctx 对应哪个 hctx

struct blk_mq_hw_ctx
  -> lock
  -> ctxs[]               映射到这个 hctx 的 ctx 集合
  -> ctx_map              哪些 ctx 里有 pending request
  -> dispatch             hctx 级别的 dispatch list
  -> tags                 driver tag / queue depth 资源
  -> queue_rq()           最终调用驱动
```

所以，一个 hctx 不是只服务一个 ctx。更常见的是：

```text
CPU0 / ctx0 --+
CPU1 / ctx1 --+--> hctx0 -> driver queue 0
CPU2 / ctx2 --+

CPU3 / ctx3 --+
CPU4 / ctx4 --+--> hctx1 -> driver queue 1
CPU5 / ctx5 --+
```

提交侧为了减少锁竞争，尽量先操作当前 CPU 的 `ctx`。这个阶段会知道目标 `hctx` 是谁，但 request 通常还没有“存到 hctx 里”。派发侧为了喂饱某个硬件队列，才会围绕一个 `hctx` 收集多个 `ctx` 或 scheduler 里的 request。

### 阶段 1：提交侧进入 ctx

一个 bio 进入 blk-mq 后，大致会先做这些事：

```text
submit_bio()
  -> blk_mq_submit_bio()
  -> 根据当前 CPU 和 I/O 类型找到 ctx，并算出目标 hctx
  -> 尝试和 plug list 里的 request merge
  -> 尝试和 scheduler / ctx / dispatch 中的 request merge
  -> merge 不上就分配 request
```

这里的“算出目标 hctx”只是一次路由选择：

```text
ctx = q->queue_ctx[cpu]
hctx = ctx->hctxs[type]
```

它回答的是：

```text
如果这个 request 后面要派发，应该运行哪个 hctx？
```

它本身不表示：

```text
request 已经进入 hctx->dispatch
request 已经拿到 driver tag
request 已经调用 queue_rq()
```

这里顺便澄清一个容易混的点：`scheduler` 和 `request merge` 不都等于 per-CPU。


| 机制                 | 粒度                          | 说明                                                     |
| ------------------ | --------------------------- | ------------------------------------------------------ |
| plug merge         | per-task                    | 当前任务自己的 plug list，最贴近提交者上下文                            |
| ctx software queue | per-CPU / per-ctx           | 没有 scheduler 或特定路径下，request 可能暂存在当前 CPU 的 `blk_mq_ctx` |
| I/O scheduler      | 通常按 request_queue / hctx 组织 | mq-deadline、bfq、kyber 等有自己的队列结构，不是简单的 per-CPU 队列       |
| dispatch list      | per-hctx                    | 已经到硬件队列派发侧，但暂时没发给驱动的 request                           |


所以 merge 机会也分层：

```text
先尝试当前任务 plug 里的 merge
再尝试 scheduler / queue 里已有 request 的 merge
也可能和 ctx software queue 或 hctx dispatch 中的 request merge
```

其中 plug 是最明显的提交侧本地优化；ctx 是 per-CPU 提交入口；scheduler 和 dispatch 更接近 queue / hctx 维度。实际能在哪里 merge，取决于是否启用 scheduler、request 当前停在哪个结构、以及内核版本的实现细节。

如果当前线程处于 plug 状态，请求可能先留在当前任务的 plug list：

```text
current->plug
  -> request A
  -> request B
```

这种情况下，它甚至还不急着进入 hctx 派发。等 `blk_finish_plug()` 或任务睡眠时，plug list 被 flush，blk-mq 才批量运行后续队列。

如果没有 plug，或者 plug flush 发生了，request 可能进入 scheduler，也可能进入 ctx 的 software queue。概念上可以画成：

```text
ctx->rq_lists[type]
  -> request A
  -> request B
  -> request C

hctx->ctx_map 标记：这个 ctx 有 pending request
```

这个时候 request 仍然是挂在 `ctx->rq_lists[type]` 上，不是挂在 `hctx->dispatch` 上。`hctx->ctx_map` 只是给派发侧一个提示：这个 hctx 关联的某个 ctx 里有活要取。

这里通常会用 `ctx->lock` 保护 per-CPU ctx 的 request list。虽然 ctx 是 per-CPU 的，但并不代表永远只有本 CPU 访问；hctx 派发侧可能会从多个 ctx 里取 request，所以仍然需要锁和标记位协调。

### 阶段 2：hctx 收集并派发

当 blk-mq 决定运行某个 hardware queue 时，主角变成 `hctx`：

```text
run hctx
  -> 看 hctx->dispatch 里是否已有待派发 request
  -> 如果没有或还能继续派发，从 hctx->ctxs[] 关联的 ctx 收集 request
  -> 根据 hctx->ctx_map 找到哪些 ctx 非空
  -> 从 ctx->rq_lists[type] 取 request
  -> 放入 hctx->dispatch 或直接尝试 issue
  -> 获取 driver tag
  -> 调用 queue_rq(hctx, bd)
```

可以近似成：

```text
for each ctx in hctx->ctxs where hctx->ctx_map says pending:
  lock(ctx)
  move requests from ctx->rq_lists[type]
  unlock(ctx)

lock(hctx)
put requests on hctx->dispatch or prepare dispatch
unlock(hctx)

while hctx has request and driver/tag resource is available:
  rq = next request
  allocate / confirm driver tag
  ret = driver->queue_rq(hctx, rq)
```

`hctx->dispatch` 的作用是保存“已经来到 hctx 派发侧，但暂时没能成功交给驱动”的 request。常见原因包括：

- driver tag / queue depth 暂时不够
- 驱动返回 `BLK_STS_RESOURCE`
- 硬件队列被 stop / quiesce
- I/O scheduler 暂时不放行

等资源恢复后，blk-mq 再 run 这个 hctx，从 dispatch list 继续派发。

### tag 什么时候分配

为了理解主线，可以先记成：

```text
request 真正要交给驱动前，需要拿到 driver tag
```

tag 代表这个 request 在 blk-mq / 驱动队列里的 in-flight 身份。驱动可以把它当成硬件 command id 或数组下标，completion 时也常靠 tag 找回 request。

但具体到内核实现，tag 分配时机和 scheduler 有关：


| 场景                   | tag 行为                                                        |
| -------------------- | ------------------------------------------------------------- |
| 无 I/O scheduler、直接派发 | request 可能较早拿到 driver tag，然后直接 `queue_rq()`                   |
| 有 mq scheduler       | 可能先拿 scheduler tag，等真正 dispatch 到驱动前再拿 driver tag             |
| 资源不足                 | 拿不到 driver tag 时，request 会停在 scheduler 或 hctx dispatch 侧，稍后重试 |


所以“阶段 2 分配 tag”是理解派发路径的好模型，但读源码时要区分 scheduler tag 和 driver tag，也要接受某些直接派发路径会更短。

### 有 scheduler 时的变化

如果启用了 mq-deadline、bfq、kyber 这类 I/O scheduler，request 不一定先进入 `ctx->rq_lists` 再被 hctx 收集。

更准确地说：

```text
提交侧
  -> request 进入 scheduler 的内部队列

hctx run
  -> 调 scheduler dispatch 回调取 request
  -> request 进入 hctx dispatch / 直接 issue
  -> queue_rq()
```

这时 scheduler 会接管排序、公平性和一部分 merge 机会；ctx 仍然负责提交侧归属和 CPU/hctx 映射，但 request 的暂存位置可能变成 scheduler 私有结构。

### 短路径：直接派发

如果没有 plug、没有复杂 scheduler、队列空闲、tag 足够，路径可以很短：

```text
submit_bio()
  -> blk_mq_submit_bio()
  -> 找到 ctx / hctx
  -> 创建 request
  -> 直接 issue 到 hctx
  -> queue_rq(hctx, bd)
```

所以不要把 `ctx->rq_lists -> hctx->dispatch -> queue_rq()` 理解成每个 I/O 必经的物理路径。它更像 blk-mq 的通用缓冲和退避机制：需要排队、合并、调度或资源不足时就会用上；能直接发时就尽量直接发。

一句话总结：

```text
ctx 解决“提交侧按 CPU 分散进入”的问题；
hctx 解决“派发侧按硬件队列集中出队”的问题。

ctx->hctxs[type] 告诉提交路径该归属哪个 hctx；
hctx->ctxs[] / ctx_map 让派发路径能反向收集这些 ctx 里的 request；
hctx->dispatch 保存已经到派发侧但暂时没发出去的 request；
最终由 hctx 调 driver queue_rq()。
```



## queue_rq() 在哪个线程里调用

`queue_rq()` 没有一个固定的专用线程。

更准确地说：

```text
谁触发 blk-mq 运行某个 hardware queue
谁就可能在自己的上下文里调用 driver 的 queue_rq()
```

常见情况有几类。

第一种是提交者线程直接派发。

如果队列很空、没有复杂 scheduler、tag 和硬件资源也够，提交 bio 的那个上下文可能一路走到驱动：

```text
writeback worker
  -> submit_bio()
  -> blk_mq_submit_bio()
  -> 分配 request
  -> run hw queue
  -> driver queue_rq()
```

这种情况下，`queue_rq()` 就是在 writeback worker 线程里被调用的。

direct I/O 也类似，可能是在用户进程的内核态调用栈里：

```text
user process
  -> write/read/direct I/O
  -> submit_bio()
  -> blk_mq_submit_bio()
  -> driver queue_rq()
```

第二种是 plug flush 的当前线程。

文件系统或 writeback 常会用 plug 批处理：

```text
blk_start_plug()
  -> submit bio A
  -> submit bio B
  -> submit bio C
blk_finish_plug()
  -> flush plug list
  -> run queue
  -> queue_rq()
```

这时前面的 `submit_bio()` 可能只是把 request 留在当前任务的 plug list 里；真正调用 `queue_rq()` 的位置可能是同一个线程稍后的 `blk_finish_plug()`。

第三种是 block layer 的异步运行队列上下文。

如果不能直接派发，或者当前上下文不适合直接派发，blk-mq 不一定马上调用 `queue_rq()`。

它可以只记录：

```text
这个 hctx 还有 request 要派发
稍后需要重新 run 这个 hardware queue
```

然后把“运行 hctx”这个动作挂到 workqueue / kblockd。后面的 worker 再进入 blk-mq 派发逻辑，并在自己的上下文里调用 `queue_rq()`。

更贴近代码的对象关系是：

```text
struct blk_mq_hw_ctx
  -> run_work          delayed_work
  -> dispatch          之前没派发出去的 request
  -> ctx_map / ctxs    有 pending request 的 software queue
  -> state             STOPPED 等状态
  -> cpumask           适合运行这个 hctx 的 CPU 集合
```

每个 `hctx` 都有一个 `run_work`。异步运行时大致是：

```text
blk_mq_delay_run_hw_queue(hctx, delay)
  -> kblockd_mod_delayed_work_on(cpu, &hctx->run_work, delay)
  -> work 进入 kblockd workqueue
```

`delay = 0` 表示尽快异步执行。资源不足时可能带一个短延迟，避免忙等。

worker 执行时再真正派发：

```text
kworker/...  // Workqueue: kblockd
  -> blk_mq_run_work_fn()
  -> blk_mq_sched_dispatch_requests(hctx)
  -> 从 hctx->dispatch / scheduler / ctx 取 request
  -> 准备 driver tag / budget
  -> q->mq_ops->queue_rq(hctx, &bd)
```

所以重点不是“有一个固定的 `queue_rq()` 线程”，而是：

```text
谁执行 hctx->run_work，
queue_rq() 就发生在哪个 worker 上下文里。
```

### 异步 run hctx 的条件

可以先记一个总条件：

```text
hctx 有 pending request
并且 queue 没有 quiesced
并且 hctx 没有 stopped
```

满足这个前提后，下面几类情况容易走异步 run：

| 场景 | 条件 | 处理 |
| --- | --- | --- |
| 调用者要求异步 | `blk_mq_run_hw_queue(hctx, true)` 或插入路径带 `run_queue_async` | 直接排 `hctx->run_work` |
| 当前 CPU 不适合 | 当前 CPU 不在 `hctx->cpumask` | 把 work 排到更合适的 CPU 上 |
| 驱动提交路径可能睡眠 | tag set / hctx 带 `BLK_MQ_F_BLOCKING` | 避免在当前上下文里直接跑可能阻塞的 `queue_rq()` |
| tag / budget 不够 | 拿不到 driver tag、scheduler budget、driver budget 等 | request 留在队列里，等资源释放后再 run |
| driver 资源不足 | `queue_rq()` 返回 `BLK_STS_RESOURCE` / `BLK_STS_DEV_RESOURCE` | 没发出去的 request 放回 `hctx->dispatch`，稍后重试 |
| driver stop 了 hctx | 驱动调用 `blk_mq_stop_hw_queue()` | run queue 跳过，直到驱动 start |
| quiesce 后恢复 | `blk_mq_unquiesce_queue()` 后还有 pending request | 通常异步 run 所有 hw queues |

注意：`stopped` 和 `quiesced` 不是“异步重试”的同义词。

```text
stopped:
  hctx 被暂停派发
  blk-mq run 到这里会跳过
  等 driver start_stopped_hw_queue()

quiesced:
  整个 queue 暂停新的 dispatch
  常用于 scheduler 切换、队列重配置、错误恢复
  等 blk_mq_unquiesce_queue()
```

恢复之后，如果还有 request，才会重新 run hctx；这个 run 常常是异步的。

### 异步派发的统一流程

不管触发点来自哪里，后面的处理大致一样：

```text
提交者 / completion / driver 恢复路径 / unquiesce
  -> 发现某个 hctx 需要重新 run
  -> blk_mq_run_hw_queue(hctx, true)
     或 blk_mq_delay_run_hw_queue(hctx, delay)
  -> hctx->run_work 进入 kblockd

kblockd worker
  -> blk_mq_run_work_fn()
  -> 检查 queue 是否 quiesced，hctx 是否 stopped
  -> 从 scheduler / ctx / hctx->dispatch 取 request
  -> 准备 tag / budget
  -> driver queue_rq(hctx, bd)
```

如果派发中途失败：

```text
queue_rq() 返回 BLK_STS_RESOURCE
  -> 当前 request 没有完成，也没有丢
  -> request 放回 hctx->dispatch
  -> hctx 标记为仍有 pending work
  -> 后面再 run hctx
```

`BLK_STS_RESOURCE` 和 `BLK_STS_DEV_RESOURCE` 都可以先理解成：

```text
现在接不住，不代表 I/O 失败。
请 block layer 稍后重试。
```

### 几个典型路径

```text
1. 提交时要求异步

submit_bio()
  -> request 进入 scheduler / ctx
  -> blk_mq_run_hw_queue(hctx, true)
  -> kblockd worker
  -> queue_rq()
```

```text
2. tag 或 driver 资源不足

run hctx
  -> queue_rq()
  -> driver 返回 BLK_STS_RESOURCE
  -> request 回到 hctx->dispatch
  -> 稍后 kblockd worker 再 run
  -> 再次 queue_rq()
```

```text
3. driver stop / start

driver queue_rq()
  -> 发现 virtqueue / submission queue 没空间
  -> blk_mq_stop_hw_queue(hctx)
  -> return BLK_STS_RESOURCE

completion / interrupt
  -> driver 发现资源恢复
  -> blk_mq_start_stopped_hw_queue(hctx, async)
  -> blk-mq 重新 run hctx
  -> queue_rq()
```

```text
4. quiesce 后恢复

blk_mq_quiesce_queue(q)
  -> 暂停新的 dispatch
  -> request 可以继续留在队列内部

稍后:
  blk_mq_unquiesce_queue(q)
  -> blk_mq_run_hw_queues(q, true)
  -> kblockd worker
  -> queue_rq()
```

驱动不是自己调用 `queue_rq()` 重试。驱动只是通过返回值、stop/start queue、completion 等方式告诉 block layer 当前能不能接 request。真正从队列里取 request 并再次调用 `queue_rq()` 的，仍然是 blk-mq。

第四种是调度器或资源恢复触发的重新派发。

这一类和第三种经常连在一起：触发点来自 scheduler 或驱动恢复路径，真正执行时仍可能落到 `hctx->run_work` / kblockd。

比如 I/O scheduler 先持有 request，等到合适时机再让 blk-mq 派发；或者驱动释放资源后调用 start / run queue 相关 helper，让 block layer 重新尝试 dispatch。

所以不要把 `queue_rq()` 理解成“必然在提交 bio 的线程里调用”。

更稳的理解是：

```text
bio -> request 的创建 / merge
  常常发生在 submit_bio() 的调用者上下文里

request -> driver queue_rq()
  可能仍在提交者上下文里
  也可能在 plug flush、block layer worker、资源恢复后的 run queue 上下文里
```

对 writeback 来说：

```text
常见快速路径:
  writeback worker 构造 bio
  writeback worker submit_bio()
  writeback worker 里创建 request
  writeback worker 里直接 queue_rq()

但不是保证:
  request 可能先排队
  后续由其他内核线程 run queue
  queue_rq() 就会在那个线程里调用
```

驱动拿到 request 后，通常会：

```text
读取 request 的 op / sector / bytes
遍历 request 里的 segment 或 bvec
把 request 映射成 scatterlist
构造 NVMe / virtio / SCSI / SATA 等设备命令
提交给硬件或虚拟后端
```

所以从驱动视角看：

```text
bio 已经被 block layer 整理成 request
驱动主要消费 request
```

从上层视角看：

```text
bio 没有消失
它被挂在 request 里
等 request 完成时再逐个 bio_endio()
```



## split 和 request 的关系

split 发生在 request 创建前更好理解。

如果原始 bio 太大或不满足限制：

```text
原始 bio: sector 1000, len 1MB
queue max request size: 256KB

block layer split:
  bio 1: sector 1000, len 256KB
  bio 2: sector 1512, len 256KB
  bio 3: sector 2024, len 256KB
  bio 4: sector 2536, len 256KB
```

这些 split 出来的 bio 会分别走 blk-mq 提交流程：

```text
每个 split bio
  -> 尝试 merge
  -> merge 失败则创建 request
```

因此不是“一个原始 bio 必然对应一个 request”，而是：

```text
一个 bio 可能被 split 成多个 bio
多个 bio 又可能被 merge 到同一个 request
最终 request 数量取决于 queue limits、merge 机会、scheduler 和当前队列状态
```



## blk-mq 关键概念速查

前面已经展开过 ctx、hctx、tag、scheduler、plug、merge、split 的细节，这里只留一个速查表，避免重复：

| 概念 | 一句话 |
| --- | --- |
| `blk_mq_ctx` | per-CPU 提交入口，减少全局锁竞争，也给短暂排队和 merge 留窗口 |
| `blk_mq_hw_ctx` / `hctx` | block layer 的硬件队列上下文，通常映射到驱动的一条提交路径 |
| tag | in-flight request 的身份，驱动完成时常用它找回 request |
| I/O scheduler | 对 request 做排序、公平性和部分 merge；`none`、`mq-deadline`、`bfq`、`kyber` 是常见选择 |
| plug | 当前任务本地的短暂批处理，`blk_finish_plug()` 时批量 flush，提高 merge 机会 |
| merge | 把兼容且相邻的 bio / request 合并，减少 request 数量和软件开销 |
| split | 按 queue limits、DMA 边界、discard / zone 等限制把 bio 拆开 |

可以通过 sysfs 观察 scheduler：

```text
cat /sys/block/<disk>/queue/scheduler
```



## stacked block device

device mapper、md raid、loop、crypt 这类都是 stacked block device。

路径可能是：

```text
文件系统
  -> bio 指向 /dev/dm-0
  -> dm 层收到 bio
  -> 查映射表
  -> clone / split / remap bio
  -> 新 bio 指向 /dev/nvme0n1 或其他下层设备
  -> submit_bio()
```

stacked device 可能是 bio-based，也可能最后走 request-based 下层队列。

这一层的核心工作是：


| 动作                | 说明                            |
| ----------------- | ----------------------------- |
| remap             | 修改 bio 的目标 `bi_bdev` 和 sector |
| clone             | 复制 bio 元信息，共享或引用原始 bvec       |
| split             | 一段上层 I/O 拆到多个下层设备             |
| chain             | 等多个下层 bio 完成后再完成上层 bio        |
| error propagation | 下层错误汇总回上层 bio                 |




## flush、FUA 和写缓存

block layer 还负责把文件系统的持久化要求转成块设备语义。

设备可能有 volatile write cache：

```text
普通 write 完成
  -> 可能只是进了设备缓存
  -> 不一定掉电不丢
```

常见机制：


| 机制             | 含义                                            |
| -------------- | --------------------------------------------- |
| `REQ_OP_FLUSH` | 要求设备把 volatile cache 刷到持久介质                   |
| `REQ_PREFLUSH` | 当前请求前需要先 flush                                |
| `REQ_FUA`      | 当前写请求本身要求 Force Unit Access                   |
| `write_cache`  | sysfs 中的设备写缓存视图，比如 write back / write through |


文件系统做 `fsync()` 时，可能会产生：

```text
数据写 bio
  -> 元数据 / journal 写 bio
  -> preflush / FUA / flush
```

实际顺序取决于文件系统、设备能力和 block layer flush machinery。

关键点：

```text
普通 write request 完成 != 数据一定持久化
flush / FUA 正确穿透到真实设备，持久化语义才可靠
```

虚拟化场景尤其要小心：guest 的 flush 必须传到 host 后端，host 后端还要正确传到底层真实存储。

## discard 和 write zeroes

block layer 不只处理 read / write。

SSD、thin provisioning、虚拟磁盘常见这些操作：


| 操作                            | 含义                         |
| ----------------------------- | -------------------------- |
| `REQ_OP_DISCARD`              | 上层告诉设备某些范围不用了，SSD 上类似 TRIM |
| `REQ_OP_WRITE_ZEROES`         | 请求设备把某段范围变成 0              |
| secure erase / secure discard | 带安全语义的擦除，取决于设备支持           |


queue limits 里会记录：

```text
discard_granularity
discard_alignment
max_discard_sectors
max_write_zeroes_sectors
```

如果设备不支持，block layer 或上层可能退化成普通写零，或者直接返回不支持。

## zoned block device

zoned block device 把设备分成 zone。

这类设备对写入顺序有限制，比如 host-managed SMR 硬盘、ZNS SSD。

block layer 会暴露和维护：


| 概念               | 说明               |
| ---------------- | ---------------- |
| zone size        | zone 大小          |
| write pointer    | 当前 zone 可写位置     |
| zone append      | 由设备选择实际写入位置并返回   |
| max open zones   | 最大 open zone 数   |
| max active zones | 最大 active zone 数 |


对普通文件系统来说，zoned device 往往需要专门支持，否则随机写可能不符合设备要求。

## integrity 和 inline encryption

block layer 还能携带额外的元信息。

### Data integrity

某些设备支持 end-to-end data integrity，比如 T10 PI。

bio 里可能带：

```text
struct bio_integrity_payload
```

它描述数据校验 / protection information。block layer 负责把 integrity payload 和数据一起传给驱动。

### Inline encryption

移动设备、UFS、eMMC、NVMe 等可能支持 inline encryption。

bio 里可能带：

```text
struct bio_crypt_ctx
```

它描述加密 keyslot、DUN 等信息。block layer 把加密上下文传给支持的驱动，驱动再配置硬件 inline crypto engine。

## cgroup、throttling 和统计

block layer 是 cgroup I/O 控制的关键位置。

它可以做：


| 机制                    | 说明                           |
| --------------------- | ---------------------------- |
| blkcg                 | block cgroup 基础设施            |
| io.max / blk-throttle | 对 cgroup 限制 IOPS / bandwidth |
| iolatency             | 控制延迟目标                       |
| iocost                | 用成本模型分配 I/O 资源               |
| BFQ cgroup            | 通过 BFQ 做更细的公平性控制             |
| iostats               | 统计读写次数、扇区数、时间、in-flight 等    |


常见观察点：

```text
/sys/block/<disk>/stat
/sys/block/<disk>/queue/iostats
/sys/fs/cgroup/io.stat
/sys/fs/cgroup/io.max
```

BIO 字段里的 `bi_blkg` 就和 blkcg 归属有关。

## polling 和 completion affinity

高速设备上，中断开销可能很高。

block layer 支持 polling：

```text
提交 I/O
  -> 用户或内核路径主动 poll completion
  -> 减少中断和调度延迟
```

典型场景：

- NVMe
- io_uring fixed file / polled I/O
- 低延迟存储

队列里还可能有 completion affinity 相关策略，比如让 completion 尽量回到提交 CPU 或同组 CPU，减少跨 CPU cache 开销。

## timeout、freeze 和 quiesce

block layer 还处理设备状态变化和错误恢复。

常见机制：


| 机制                    | 作用                                       |
| --------------------- | ---------------------------------------- |
| request timeout       | request 超时后调用驱动 `timeout()` 回调           |
| quiesce queue         | 停止新的 dispatch，等待正在 dispatch 的路径退出        |
| freeze queue          | 冻结队列，阻止新的 request 进入，常用于设备移除、重配置、suspend |
| stop / start hw queue | 驱动临时停止或恢复某个硬件队列                          |
| requeue               | request 没法提交时放回队列稍后重试                    |


比如设备队列满了：

```text
driver queue_rq()
  -> 硬件 submission queue 没空间
  -> return BLK_STS_RESOURCE
  -> block layer 稍后重新运行队列
```

或者驱动主动：

```text
blk_mq_stop_hw_queue(hctx)
  -> 暂停这个 hctx 派发
设备有空间后
  -> blk_mq_start_stopped_hw_queues(q, true)
```



## 提交路径

一次普通 writeback I/O 可以这样看：

```text
writeback worker
  -> 文件系统 writepages
  -> 查 extent / block map
  -> 构造 bio
  -> bio_add_folio() / bio_add_page()
  -> submit_bio()
```

进入 block layer 后：

```text
submit_bio()
  -> submit_bio_noacct()
  -> 检查 bio 参数、统计和 cgroup 归属
  -> 如果是 partition，换算到底层 disk sector
  -> 如果是 stacked device，可能进入上层设备自己的 submit_bio/remap 路径
  -> 根据 queue limits split
  -> blk_mq_submit_bio()
  -> 尝试 plug / merge
  -> 分配 request 和 tag
  -> 插入 scheduler 或 software queue
  -> 运行 hardware queue
  -> 调用 driver queue_rq()
```

`submit_bio()` 返回通常只表示 bio 已经进入块层提交路径，不表示 I/O 已经完成，也不表示数据已经持久化。

```text
submit_bio() 返回
  != 设备完成
  != bio->bi_end_io() 已经调用
  != 数据掉电不丢
```



## 完成路径

完成路径通常和提交路径不是同一个上下文。

提交者可能是：

- 用户进程 direct I/O
- `fsync()` 进程
- writeback `kworker`
- swap 路径
- 文件系统内部线程

完成者可能是：

- 硬中断
- softirq / block completion 路径
- NVMe polling
- virtio callback
- 某个内核线程或虚拟化后端通知路径

典型完成路径：

```text
设备完成命令
  -> 驱动拿到 completion
  -> 找到 request
  -> blk_mq_end_request(rq, status)
  -> block layer 完成 request
  -> 遍历 request 里的 bio
  -> bio_endio(bio)
  -> bio->bi_status = status
  -> bio->bi_end_io(bio)
  -> 上层清 page writeback / 设置 uptodate / 唤醒等待者
```

如果是 partial completion：

```text
blk_update_request(rq, status, nr_bytes)
  -> 完成 request 的一部分
  -> 推进 request / bio iterator
  -> 如果还有剩余，继续处理剩余部分
```

block layer 和设备协议通常不保证 request 按提交顺序完成。上层必须能处理乱序 completion。

## block layer 给驱动暴露的接口

现代块设备驱动大多走 blk-mq。

驱动要做两类事情：

```text
注册一个 block device
实现 blk_mq_ops，让 block layer 能把 request 交给驱动
```



## 驱动注册一个块设备

一个简化的 blk-mq 块设备驱动初始化路径：

```text
准备 struct blk_mq_tag_set
  -> tag_set.ops = &my_mq_ops
  -> tag_set.nr_hw_queues = 硬件队列数
  -> tag_set.queue_depth = 每个硬件队列可同时 outstanding 的 request 数
  -> tag_set.cmd_size = 每个 request 后面附带的驱动私有命令大小
  -> tag_set.numa_node = NUMA node
  -> tag_set.flags = BLK_MQ_F_*
  -> tag_set.driver_data = driver object

blk_mq_alloc_tag_set(&tag_set)

blk_mq_alloc_disk(&tag_set, queuedata)
  -> 得到 gendisk 和 request_queue

设置 gendisk 名字、容量、fops、private_data
设置 queue limits
add_disk()
```

大概对象关系：

```text
driver device object
  -> blk_mq_tag_set
       -> blk_mq_ops
       -> queue maps
       -> tags
  -> gendisk
       -> request_queue
       -> block_device_operations
```

`gendisk` 面向系统暴露磁盘：


| 字段 / 概念                 | 作用                                              |
| ----------------------- | ----------------------------------------------- |
| `disk_name`             | `/sys/block` 和 `/dev` 里看到的名字，比如 `vda`、`nvme0n1` |
| `major` / `first_minor` | 设备号                                             |
| `minors`                | 分区 minor 范围                                     |
| `fops`                  | `struct block_device_operations`                |
| `private_data`          | 驱动私有对象                                          |
| capacity                | 设备容量，通常用 `set_capacity()` 设置                    |
| `queue`                 | 对应 `request_queue`                              |


`block_device_operations` 更多是块设备文件层面的操作：


| 回调             | 作用                                                           |
| -------------- | ------------------------------------------------------------ |
| `open`         | 块设备打开                                                        |
| `release`      | 块设备关闭                                                        |
| `ioctl`        | 设备 ioctl                                                     |
| `check_events` | 介质变化等事件检查                                                    |
| `getgeo`       | 老式几何信息                                                       |
| `submit_bio`   | bio-based 驱动或 stacked driver 可直接处理 bio，request-based 驱动通常不靠它 |


真正的数据 I/O 主路径，blk-mq 驱动主要看 `blk_mq_ops`。

## `struct blk_mq_ops`

`blk_mq_ops` 是 block layer 和 request-based 块设备驱动之间最核心的通信接口。

常见回调：


| 回调                                           | 方向                       | 作用                                   |
| -------------------------------------------- | ------------------------ | ------------------------------------ |
| `queue_rq(hctx, bd)`                         | block layer -> driver    | 把一个 request 交给驱动提交                   |
| `queue_rqs(rqlist)`                          | block layer -> driver    | 批量提交一组 request，驱动可选实现                |
| `commit_rqs(hctx)`                           | block layer -> driver    | 批量提交时最后 kick 硬件                      |
| `map_queues(set)`                            | driver -> block layer 配置 | 驱动指定 CPU 到 hardware queue 的映射        |
| `init_hctx(hctx, driver_data, hctx_idx)`     | 初始化                      | block layer 建好 hctx 后，驱动初始化对应硬件队列上下文 |
| `exit_hctx(hctx, hctx_idx)`                  | 清理                       | 释放 hctx 相关资源                         |
| `init_request(set, rq, hctx_idx, numa_node)` | 初始化                      | 每个 request 分配时初始化驱动私有数据              |
| `exit_request(set, rq, hctx_idx)`            | 清理                       | request 释放时清理驱动私有数据                  |
| `cleanup_rq(rq)`                             | 清理                       | request 未正常完成前释放时清理                  |
| `timeout(rq)`                                | block layer -> driver    | request 超时时让驱动决定恢复策略                 |
| `poll(hctx, batch)`                          | block layer -> driver    | polling 模式下主动查询 completion           |
| `complete(rq)`                               | completion               | 驱动可自定义 request completion 处理         |
| `busy(q)`                                    | block layer 查询           | 判断队列是否忙                              |
| `get_budget` / `put_budget`                  | 资源控制                     | 驱动私有 budget 管理                       |


最关键的是 `queue_rq()`。

## `queue_rq()` 的语义

block layer 调用：

```c
blk_status_t queue_rq(struct blk_mq_hw_ctx *hctx,
                      const struct blk_mq_queue_data *bd);
```

`bd->rq` 就是要提交的 request。

驱动一般做：

```text
queue_rq(hctx, bd)
  -> rq = bd->rq
  -> blk_mq_start_request(rq)
  -> 根据 req_op(rq) 判断 read/write/flush/discard
  -> 用 blk_rq_map_sg() 或 rq_for_each_*() 构造数据描述
  -> 构造设备协议命令
  -> 把命令放入硬件 queue / virtqueue / DMA ring
  -> 如果需要，bd->last 时 kick 硬件
  -> return BLK_STS_OK
```

常见返回值：


| 返回值                    | 含义                             |
| ---------------------- | ------------------------------ |
| `BLK_STS_OK`           | 驱动接受了 request，之后会完成它           |
| `BLK_STS_RESOURCE`     | 驱动 / 队列资源暂时不足，block layer 稍后重试 |
| `BLK_STS_DEV_RESOURCE` | 设备资源暂时不足                       |
| `BLK_STS_IOERR` 等错误    | request 失败                     |


如果驱动成功接收 request，它必须最终完成它：

```text
blk_mq_end_request(rq, status)
```

如果只是完成一部分：

```text
more = blk_update_request(rq, status, nr_bytes)
if (!more)
    blk_mq_end_request(rq, status)
```

实际代码会按驱动和内核版本有不同封装，但语义是：

```text
block layer 把 request 交给驱动
驱动接受后负责在完成路径把 request 还给 block layer
```



## 驱动如何读取 request

驱动通常关心：


| 想知道什么              | 常见 helper                                      |
| ------------------ | ---------------------------------------------- |
| 操作类型               | `req_op(rq)`                                   |
| 读写方向               | `rq_data_dir(rq)`                              |
| 起始 sector          | `blk_rq_pos(rq)`                               |
| 剩余字节数              | `blk_rq_bytes(rq)`                             |
| request 总 sector 数 | `blk_rq_sectors(rq)`                           |
| 是否 passthrough     | `blk_rq_is_passthrough(rq)`                    |
| 遍历数据段              | `rq_for_each_segment()` / `rq_for_each_bvec()` |
| 构造 scatterlist     | `blk_rq_map_sg(q, rq, sglist)`                 |
| 驱动私有命令区            | `blk_mq_rq_to_pdu(rq)`                         |
| tag                | `rq->tag` 或相关 helper                           |


一个典型数据请求在驱动里会变成：

```text
request
  -> req_op / sector / bytes
  -> scatterlist
  -> dma_map_sg()
  -> device command
  -> submit to hardware queue
```

对 NVMe：

```text
request
  -> NVMe command
  -> PRP / SGL 指向 DMA buffer
  -> submission queue
  -> doorbell
```

对 virtio-blk：

```text
request
  -> virtio_blk_outhdr
  -> data scatterlist
  -> status byte
  -> virtqueue descriptor chain
  -> virtqueue kick
```



## 驱动如何通知完成

设备完成后，驱动要通知 block layer。

典型路径：

```text
硬件 interrupt / polling
  -> 驱动读取 completion entry
  -> 根据 queue id + tag 找到 request
  -> 解除 DMA 映射
  -> blk_mq_end_request(rq, status)
```

block layer 之后会：

```text
完成 request
  -> 完成 request 中的 bio
  -> 调 bio_endio()
  -> 调 bio->bi_end_io()
  -> 释放 request tag
  -> 唤醒等待者
  -> 更新统计
```

驱动也可能用 batch completion 或 poll completion，具体取决于设备和内核版本。

## 驱动和 block layer 的双向通信

可以把通信分成四条线：


| 方向                         | 机制                                                       | 说明                     |
| -------------------------- | -------------------------------------------------------- | ---------------------- |
| 上层 -> block layer          | `submit_bio()`                                           | 文件系统等提交 bio            |
| block layer -> driver      | `blk_mq_ops.queue_rq()`                                  | block layer 派发 request |
| driver -> block layer      | `blk_mq_end_request()` / completion APIs                 | 驱动报告完成或错误              |
| driver -> block layer 状态反馈 | return status、stop/start queue、timeout、poll、queue limits | 驱动告诉块层资源、能力和状态         |


更细一点：

```text
能力沟通：
  queue limits
  tag_set.nr_hw_queues
  tag_set.queue_depth
  blk_mq_ops.map_queues()

提交沟通：
  queue_rq()
  queue_rqs()
  commit_rqs()

背压沟通：
  BLK_STS_RESOURCE
  blk_mq_stop_hw_queue()
  blk_mq_start_stopped_hw_queues()
  blk_mq_delay_run_hw_queue()

完成沟通：
  blk_mq_end_request()
  blk_update_request()
  blk_mq_complete_request()

错误恢复：
  timeout()
  requeue
  quiesce / freeze
```



## bio-based 和 request-based 驱动

不是所有块设备层都必须以 request 作为主接口。

可以粗略分两类：


| 类型                     | 接口           | 典型对象                               |
| ---------------------- | ------------ | ---------------------------------- |
| bio-based              | 直接处理或重映射 bio | dm、某些 stacked target、简单虚拟层         |
| request-based / blk-mq | 处理 request   | NVMe、virtio-blk、SCSI、SATA、多数真实设备驱动 |


bio-based 适合做映射：

```text
收到上层 bio
  -> 改 bi_bdev / bi_sector
  -> clone / split
  -> submit 到下层设备
```

request-based 适合真实设备提交：

```text
收到 request
  -> 构造设备命令
  -> 提交到硬件
```

很多路径会混合：

```text
文件系统
  -> bio
  -> dm-crypt bio-based remap
  -> 下层 nvme request-based blk-mq
  -> NVMe command
```



## 例子一：virtio-blk

virtio-blk 是虚拟机里常见的块设备。

从 guest Linux 看：

```text
guest 文件系统 / page cache
  -> bio
  -> submit_bio()
  -> guest block layer
  -> blk-mq request
  -> virtio-blk queue_rq()
  -> virtqueue descriptor chain
  -> kick host
```

virtio-blk 驱动会把 request 翻译成：

```text
virtio_blk_outhdr
  -> type: read / write / flush / discard
  -> sector

data sg
  -> request 对应的内存页

status byte
  -> host 完成后写状态

virtqueue descriptor chain
  -> out header + data + status
```

写路径：

```text
guest 内存 page
  -> bio_vec
  -> request segment
  -> virtio scatterlist
  -> host 通过 virtqueue 访问 guest memory
  -> host backend 写到底层文件 / 块设备 / 网络存储
```

完成路径：

```text
host backend 完成
  -> 写 virtio status
  -> used ring
  -> guest interrupt / callback
  -> virtblk_done()
  -> blk_mq_end_request()
  -> bio_endio()
```

这里 block layer 和驱动的分界很清楚：

```text
block layer 给 virtio-blk 的是 request
virtio-blk 给 host 的是 virtqueue descriptor
```



## 例子二：NVMe

NVMe 是天然多队列设备，和 blk-mq 很贴合。

从 Linux 内核看：

```text
文件系统 / direct I/O / swap
  -> bio
  -> submit_bio()
  -> blk-mq request
  -> nvme_queue_rq()
  -> NVMe command
  -> PRP / SGL 指向数据页
  -> submission queue
  -> doorbell
```

NVMe 驱动把 request 信息映射到 NVMe command：


| request 信息                     | NVMe 命令概念                    |
| ------------------------------ | ---------------------------- |
| `bi_bdev` / disk namespace     | namespace id                 |
| `blk_rq_pos(rq)`               | 起始 LBA                       |
| `blk_rq_bytes(rq)`             | block count                  |
| request segments               | PRP list 或 SGL               |
| `REQ_OP_READ` / `REQ_OP_WRITE` | NVMe read / write opcode     |
| flush / FUA flags              | NVMe flush command 或 FUA 控制位 |


完成路径：

```text
controller 写 completion queue entry
  -> MSI-X interrupt 或 polling
  -> nvme completion path
  -> 根据 command id / tag 找 request
  -> blk_mq_end_request()
  -> bio_endio()
```

NVMe 的重点：

- 多个 CPU 可以映射到多个 hardware queue
- tag 可以直接对应命令标识
- PRP / SGL 能描述分散的 DMA buffer
- polling 可以降低低延迟场景的中断开销
- flush / FUA 对持久化延迟影响很大



## buffered I/O、direct I/O 和 block layer

普通 buffered write：

```text
write()
  -> copy_from_user 到 page cache
  -> 标记 dirty
  -> 后台 writeback
  -> 文件系统构造 bio
  -> block layer
```

普通 buffered read：

```text
read()
  -> page cache miss
  -> 文件系统构造 read bio
  -> block layer
  -> I/O 完成后 page uptodate
  -> copy_to_user
```

direct I/O：

```text
用户 buffer pin page
  -> 文件系统映射 file offset 到 sector
  -> bio_vec 指向用户页
  -> block layer
  -> 完成后唤醒 direct I/O 等待者
```

所以 block layer 不知道“这是哪个文件的第几个字节”。它处理的是：

```text
block device + sector range + memory pages + operation
```



## 错误传播

错误大概这样往上传：

```text
设备 / 驱动发现错误
  -> request status
  -> blk_mq_end_request(rq, status)
  -> bio->bi_status
  -> bio->bi_end_io()
  -> 文件系统 / direct I/O / page cache 记录错误
  -> fsync() / read() / write() / io_uring completion 返回错误
```

对 buffered write 来说，普通 `write()` 可能早已返回。后台 writeback 的错误通常记录到 mapping / inode，后续 `fsync()`、`close()` 或其他同步点再报告。

## 一个完整心智模型

完整生命周期可以压成一条线：

```text
bio alloc / fill
  -> submit_bio()
  -> split / remap / merge
  -> request allocate / dispatch
  -> queue_rq()
  -> device command
  -> completion
  -> blk_mq_end_request()
  -> bio_endio()
```

可以把 block layer 记成四句话：

```text
上层用 bio 描述“对哪个块设备的哪段 sector 做 I/O，数据在哪些页里”。

block layer 根据队列限制、调度策略和 cgroup 状态，把 bio 拆分、合并、排队，形成 request。

blk-mq 把 request 从 per-CPU software queue 派发到 hardware queue context，再调用驱动 queue_rq()。

驱动把 request 翻译成设备命令；完成后再通过 block completion path 回到 bio->bi_end_io()。
```

再简化：

```text
bio      面向上层
request  面向 block layer 和驱动
command  面向设备
```

这三个层次分清楚，Linux 块 I/O 的主路径就顺了。

## 读源码入口

不同内核版本函数细节会变，但读代码时可以先沿这些名字找：

| 想看什么 | 入口 |
| --- | --- |
| 上层提交 bio | `submit_bio()` / `submit_bio_noacct()` |
| bio 进入 blk-mq | `blk_mq_submit_bio()` |
| 拆分和队列限制 | `bio_split_to_limits()` / queue limits 相关 helper |
| ctx / hctx 映射 | `blk_mq_map_queue()` / `blk_mq_map_swqueue()` |
| 运行硬件队列 | `blk_mq_run_hw_queue()` / `blk_mq_delay_run_hw_queue()` |
| scheduler 派发 | `blk_mq_sched_dispatch_requests()` |
| 交给驱动 | `blk_mq_ops.queue_rq()` |
| 驱动完成 request | `blk_mq_end_request()` / `blk_update_request()` |
| 回到 bio | `bio_endio()` / `bio->bi_end_io()` |
