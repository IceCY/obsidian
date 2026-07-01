# virtio-blk 设备驱动机制

先抓住一句话：

`virtio-blk` 是 guest Linux 里的块设备驱动。它从 guest block layer 收到 `request`，把 request 翻译成 virtio block 协议命令，挂到 virtqueue 里，然后 kick host；host 后端完成 I/O 后，把结果写回 used ring 并触发 guest completion，guest 驱动再调用 block layer 的完成接口。

整体路径：

```text
guest 文件系统 / page cache
  -> bio
  -> submit_bio()
  -> block layer / blk-mq
  -> request
  -> virtio-blk queue_rq()
  -> virtqueue descriptor chain
  -> kick host
  -> host vhost / QEMU / hypervisor backend
  -> host 文件 / 块设备 / 网络存储
  -> used ring / interrupt
  -> guest virtblk_done()
  -> blk_mq_end_request()
  -> bio_endio()
```

这里要分清两个边界：

```text
block layer 给 virtio-blk 的是 request
virtio-blk 给 virtio device 的是 descriptor chain
host 后端真正访问的是 guest memory 里的 descriptor 指向的数据页
```

## virtio-blk 在哪一层

从 guest 内核看，`virtio-blk` 是一个普通块设备驱动：

```text
/dev/vda
  -> struct gendisk
  -> struct request_queue
  -> blk-mq tag_set
  -> virtio-blk driver
```

它对上暴露的是 Linux block device，对下使用的是 virtio transport。

virtio transport 可以是：

| transport | 常见场景 | 说明 |
| --- | --- | --- |
| virtio-pci | KVM / QEMU 虚拟机里最常见 | guest 通过 PCI 发现 virtio 设备，配置 MSI-X、中断、virtqueue |
| virtio-mmio | 嵌入式、简单虚拟平台 | 通过 MMIO 寄存器发现和控制 virtio 设备 |
| virtio-ccw | s390 | IBM mainframe 相关 |

`virtio-blk` 只关心“我是一个 virtio block device”。底层 transport 负责发现设备、读写配置空间、设置队列地址、触发通知和接收中断。

可以这么切：

```text
blk-mq
  -> 负责 request、tag、queue limit、调度、完成传播

virtio-blk
  -> 负责把 request 变成 virtio block 命令

virtio core / transport
  -> 负责 virtqueue、feature negotiation、notify、interrupt

host backend
  -> 负责真正执行 I/O
```

## 初始化主线

guest 发现 virtio block 设备后，大致初始化路径是：

```text
virtio transport 发现设备
  -> 匹配 virtio_blk driver
  -> probe()
  -> feature negotiation
  -> 读取设备容量和队列能力
  -> 创建 virtqueue
  -> 准备 blk_mq_tag_set
  -> 创建 request_queue / gendisk
  -> 设置 queue limits
  -> add_disk()
  -> 用户态看到 /dev/vda
```

几个关键对象：

| 对象 | 作用 |
| --- | --- |
| `struct virtio_device` | virtio core 暴露给具体驱动的设备对象 |
| `struct virtio_blk` | virtio-blk 驱动自己的主设备结构，保存 disk、tag_set、virtqueue 信息等 |
| `struct virtio_blk_vq` | virtio-blk 对一个 virtqueue 的封装，通常对应一个 blk-mq hardware queue |
| `struct request_queue` | block layer 的队列入口 |
| `struct gendisk` | Linux 对一个磁盘的抽象，决定 `/sys/block/vda`、`/dev/vda` 等 |
| `struct blk_mq_tag_set` | 驱动向 blk-mq 声明队列数量、队列深度、私有数据大小和回调 |
| `struct virtqueue` | virtio core 的队列对象，对应 split ring 或 packed ring |

初始化时最重要的是两件事：

1. 和设备协商能力。
2. 把 blk-mq 的 hardware queue 映射到 virtqueue。

## feature negotiation

virtio 设备不会假设 guest 和 host 支持完全一样的能力。驱动初始化时会先做 feature negotiation。

流程大概是：

```text
host 暴露 device features
  -> guest driver 选择自己支持且愿意启用的 features
  -> 写回 accepted features
  -> virtio core 完成 FEATURES_OK
  -> 后续驱动按协商结果使用设备
```

virtio-blk 常见能力包括：

| feature | 含义 |
| --- | --- |
| `VIRTIO_BLK_F_SIZE_MAX` | 单个 segment 最大大小 |
| `VIRTIO_BLK_F_SEG_MAX` | 一个请求最多多少个 segment |
| `VIRTIO_BLK_F_GEOMETRY` | 传统 CHS 几何信息，现代系统通常不重要 |
| `VIRTIO_BLK_F_RO` | 只读设备 |
| `VIRTIO_BLK_F_BLK_SIZE` | 设备报告 logical block size |
| `VIRTIO_BLK_F_FLUSH` | 支持 flush |
| `VIRTIO_BLK_F_TOPOLOGY` | 报告物理块、对齐、最优 I/O size 等拓扑信息 |
| `VIRTIO_BLK_F_CONFIG_WCE` | guest 可读写 write cache enable 配置 |
| `VIRTIO_BLK_F_MQ` | 支持多 virtqueue |
| `VIRTIO_BLK_F_DISCARD` | 支持 discard / trim |
| `VIRTIO_BLK_F_WRITE_ZEROES` | 支持 write zeroes |

这些 feature 会反向影响 block layer 的 queue limits 和能力暴露：

```text
virtio config
  -> logical block size
  -> max sectors
  -> max segments
  -> discard granularity / max discard sectors
  -> write cache / flush capability
  -> queue count
  -> request_queue limits
```

比如设备告诉 guest 最多支持 `seg_max` 个 segment，驱动就不能让 block layer 交给自己一个超过这个限制的 request。正确做法是把限制设置到 `request_queue`，让 block layer 提前拆分。

## 设备容量和 block size

virtio-blk 配置空间里会报告容量，单位通常是 512-byte sector：

```text
capacity = number of 512-byte sectors
```

这也是为什么块层里经常看到 sector 概念。即使设备 logical block size 是 4 KiB，很多接口里地址仍然以 512-byte sector 表示。

驱动初始化时会设置：

```text
disk capacity
queue logical block size
queue physical block size
queue io min / io opt
queue max sectors
queue max segments
```

对上层来说，这些信息最后会出现在：

```text
/sys/block/vda/size
/sys/block/vda/queue/logical_block_size
/sys/block/vda/queue/physical_block_size
/sys/block/vda/queue/max_sectors_kb
/sys/block/vda/queue/max_segments
/sys/block/vda/queue/write_cache
```

## blk-mq 和 virtqueue 的映射

现代 virtio-blk 走 blk-mq。可以把一个 blk-mq hardware queue 理解成一个提交通道：

```text
blk_mq_hw_ctx
  -> virtio_blk_vq
  -> virtqueue
```

完整一点，其实有两层映射：

```text
第一层：blk-mq 建立

CPU / blk_mq_ctx
  -> blk_mq_hw_ctx

第二层：virtio-blk 驱动建立

blk_mq_hw_ctx
  -> virtio_blk_vq
  -> virtqueue
```

第一层是 block layer 的事情。它靠 `blk_mq_tag_set` 里的 queue map 建 CPU 到 hctx 的路由：

```text
tag_set->map[type].mq_map[cpu] = hctx_index
request_queue->queue_hw_ctx[hctx_index] = hctx
blk_mq_ctx->hctxs[type] = hctx
```

第二层是 virtio-blk 的事情。驱动会把 virtio core 创建出来的 virtqueue 保存到自己的队列数组里，再把每个 blk-mq hctx 绑定到其中一个队列对象：

```text
struct virtio_blk
  -> vqs[]              一组 virtio_blk_vq
     -> vq              真正的 struct virtqueue

struct blk_mq_hw_ctx
  -> driver_data        指向对应的 virtio_blk_vq
```

可以把 `hctx->driver_data` 理解成 block layer 留给驱动的“队列私有指针”。block layer 不知道里面是什么；对 virtio-blk 来说，它通常就是“这个 hctx 对应哪个 virtqueue”的答案。

如果设备只支持一个 virtqueue：

```text
所有 CPU 的 blk_mq_ctx
  -> 同一个 blk_mq_hw_ctx
  -> 同一个 virtqueue
```

如果设备支持 `VIRTIO_BLK_F_MQ`：

```text
CPU 0,1  -> hctx 0 -> virtqueue 0
CPU 2,3  -> hctx 1 -> virtqueue 1
CPU 4,5  -> hctx 2 -> virtqueue 2
...
```

实际映射由 blk-mq 的 queue map 决定，驱动会根据设备队列数量、CPU 数量、MSI-X vector 等信息设置 `tag_set.nr_hw_queues` 和映射关系。

初始化时可以按这个顺序理解：

```text
virtio-blk probe()
  -> 读取设备支持几个 queue
  -> virtio core 创建 virtqueue0..N
  -> virtio-blk 保存到 vblk->vqs[i].vq

准备 blk_mq_tag_set
  -> tag_set.nr_hw_queues = virtqueue 数量
  -> tag_set.ops = virtblk_mq_ops
  -> tag_set.driver_data = vblk

blk-mq 初始化 request_queue
  -> 根据 nr_hw_queues 创建 hctx0..N
  -> 根据 queue map 建 CPU/ctx -> hctx
  -> 调用 virtio-blk 的 init_hctx(hctx, vblk, hctx_idx)

virtio-blk init_hctx()
  -> hctx->driver_data = &vblk->vqs[hctx_idx]
```

所以这里有两个不同的“编号”：

| 编号 | 谁使用 | 含义 |
| --- | --- | --- |
| `cpu` | blk-mq | 当前提交 I/O 的 CPU |
| `hctx_index` | blk-mq / 驱动共享 | blk-mq 硬件队列编号，virtio-blk 通常用它索引 `vblk->vqs[]` |
| `virtqueue index` | virtio-blk / virtio core | virtio 设备的队列编号，通常和 `hctx_index` 一一对应 |

常见情况下是：

```text
hctx0 -> vblk->vqs[0] -> virtqueue0
hctx1 -> vblk->vqs[1] -> virtqueue1
hctx2 -> vblk->vqs[2] -> virtqueue2
```

但概念上不要把 `blk_mq_ctx`、`blk_mq_hw_ctx` 和 `virtqueue` 混成同一个东西：

```text
blk_mq_ctx       CPU 本地提交入口，数量通常接近 CPU 数
blk_mq_hw_ctx    block layer 的派发队列，数量由 tag_set.nr_hw_queues 决定
virtqueue        virtio 协议里的共享 ring，数量由设备能力和驱动选择决定
```

这样做的好处是：

- 减少多个 CPU 抢同一把队列锁。
- 让多核 guest 可以并行提交 I/O。
- host 侧可以并行处理多个 virtqueue。
- completion 可以更接近提交 CPU，降低 cache 抖动。

但它不是魔法。性能还取决于 host 后端、vhost 线程、底层盘、I/O scheduler、cgroup、NUMA 和队列深度。

提交时的查找流程可以串成这样：

```text
当前 I/O 在 CPU 5 提交
  -> blk-mq 找到 q->queue_ctx[5]
  -> 根据普通 I/O queue map 找到 hctx1
  -> 调用 virtblk_queue_rq(hctx1, bd)
  -> virtio-blk 读取 hctx1->driver_data
  -> 得到 virtio_blk_vq1
  -> 取 virtio_blk_vq1->vq，也就是 virtqueue1
  -> 把 request 编成 descriptor chain 放进 virtqueue1
```

伪代码大概是：

```text
blk-mq:
  ctx = q->queue_ctx[cpu]
  hctx = ctx->hctxs[HCTX_TYPE_DEFAULT]
  queue_rq(hctx, bd)

virtio-blk:
  vblk_vq = hctx->driver_data
  vq = vblk_vq->vq
  virtqueue_add_sgs(vq, ...)
  virtqueue_kick(vq)
```

所以第一层查找解决的是：

```text
这个 CPU 提交的 request 应该交给哪个 blk_mq_hw_ctx？
```

第二层查找解决的是：

```text
这个 blk_mq_hw_ctx 在 virtio-blk 驱动里对应哪个 virtqueue？
```

## virtio block 请求格式

virtio-blk 的一个请求通常由三段组成：

```text
virtio_blk_outhdr
  -> 设备可读，guest 写给 device

data buffer
  -> read 时 device 写入 guest memory
  -> write 时 device 读取 guest memory

status byte
  -> device 写回完成状态
```

典型 descriptor chain：

```text
write request:

[outhdr: device read]
  -> [data pages: device read]
  -> [status: device write]

read request:

[outhdr: device read]
  -> [data pages: device write]
  -> [status: device write]

flush request:

[outhdr: device read]
  -> [status: device write]
```

`virtio_blk_outhdr` 里核心字段可以理解成：

| 字段 | 含义 |
| --- | --- |
| `type` | 请求类型，比如 read、write、flush、discard、write zeroes |
| `sector` | 起始 sector |
| `ioprio` | 旧接口里的 I/O priority 字段，现代路径不一定作为重点 |

status byte 常见结果：

| status | 含义 |
| --- | --- |
| `VIRTIO_BLK_S_OK` | 成功 |
| `VIRTIO_BLK_S_IOERR` | I/O 错误 |
| `VIRTIO_BLK_S_UNSUPP` | 不支持的命令 |

驱动要把这些 status 转成 block layer 能理解的 `blk_status_t`，再结束 request。

## descriptor chain 怎么描述数据

block layer 给驱动的是 `struct request`。request 里面可能挂着一个或多个 bio，每个 bio 又有若干 `bio_vec`：

```text
request
  -> bio list
     -> bio_vec: page + offset + len
     -> bio_vec: page + offset + len
```

virtio-blk 不会把这些数据复制到一个连续 buffer。它会把 request 映射成 scatterlist，再交给 virtqueue：

```text
request
  -> blk_rq_map_sg()
  -> scatterlist[]
  -> virtqueue_add_sgs()
  -> descriptor chain
```

descriptor 本质上就是告诉设备：

```text
addr = guest physical address / DMA address
len  = 这段 buffer 的长度
flag = device read 还是 device write
next = 下一个 descriptor
```

对写请求：

```text
guest page cache page
  -> bio_vec
  -> scatterlist
  -> virtqueue descriptor
  -> host 读 guest memory
  -> host 写到底层存储
```

对读请求：

```text
host 从底层存储读数据
  -> host 写 guest memory
  -> guest page cache page 变成 uptodate
```

这里的“DMA”在虚拟化里要小心理解。对 guest 来说，这是设备 DMA；对 host 来说，可能是 QEMU 进程拷贝、vhost 内核线程访问 guest memory、IOMMU 映射、硬件 vDPA，或者它们的组合。virtio 抽象隐藏了后端实现，但 guest 驱动仍然要按 DMA / scatter-gather 的规则准备内存。

## split ring 的基本结构

virtqueue 经典实现是 split virtqueue，分成三块：

```text
descriptor table
  -> guest 提供一组 descriptor 槽位

avail ring
  -> guest 告诉 device：哪些 descriptor chain 可以处理

used ring
  -> device 告诉 guest：哪些 descriptor chain 已经处理完
```

提交时：

```text
guest driver 分配 descriptor
  -> 填 descriptor table
  -> 把 chain head 放进 avail ring
  -> 更新 avail idx
  -> 内存屏障
  -> notify / kick device
```

完成时：

```text
device 处理 descriptor chain
  -> 写 status byte / data buffer
  -> 把 chain head 放进 used ring
  -> 更新 used idx
  -> 触发 interrupt
```

guest interrupt callback 里再从 used ring 取回完成的请求。

packed ring 是另一种更紧凑的 virtqueue 格式，把 descriptor、avail、used 信息合在一组 ring 里，减少 cache miss 和内存占用。驱动通常通过 virtio core 的统一接口使用 virtqueue，不需要在 virtio-blk 主路径里手写 split / packed 的细节。

## driver 和 device 怎么交互

virtio-blk 的 driver / device 交互不是“函数互相调用”，而是围绕三类东西进行：

```text
共享内存:
  virtqueue ring + descriptor 指向的数据页

doorbell / notify:
  guest driver 告诉 device：队列里有新请求

interrupt / callback:
  device 告诉 guest driver：有请求完成了
```

从 driver 视角看：

```text
准备 request 私有数据
  -> 准备 header / data pages / status byte
  -> 把这些 buffer 的地址和方向写成 descriptor chain
  -> 把 chain head 发布到 avail ring
  -> notify device
  -> 等 completion callback 或 polling
```

从 device 视角看：

```text
收到 notify，或者自己轮询 virtqueue
  -> 读取 avail ring
  -> 找到新的 descriptor chain
  -> 读取 out buffer，比如 virtio_blk_outhdr 和 write data
  -> 写入 in buffer，比如 read data 和 status byte
  -> 把完成项写到 used ring
  -> 按需触发 guest interrupt
```

所以 virtio-blk 的“命令提交”本质是：driver 把一个请求编码进 guest memory 里的 virtqueue，device 按 virtio 规范读取这段共享内存。`notify` 和 `interrupt` 只负责提示对方“该看 ring 了”，真正的请求内容和数据不在通知里传递。

## 提交路径：queue_rq()

block layer 派发 request 时，会调用 virtio-blk 注册的 `blk_mq_ops.queue_rq()`。

主线可以理解为：

```text
virtblk_queue_rq(hctx, bd)
  -> rq = bd->rq
  -> vblk_vq = hctx->driver_data
  -> vq = vblk_vq->vq
  -> blk_mq_start_request(rq)
  -> 从 request private data 取 virtblk_req
  -> 填 virtio_blk_outhdr
  -> 准备 status byte
  -> 把 request data 映射成 scatterlist
  -> virtqueue_add_sgs(vq, sgs, out_num, in_num, vbr)
  -> virtqueue_kick_prepare()
  -> virtqueue_notify()
  -> 返回 BLK_STS_OK
```

也就是说，`queue_rq()` 收到的不是 virtqueue 本身，而是 block layer 的 `hctx`。virtio-blk 在初始化 `hctx` 时已经把 `hctx->driver_data` 指向了自己的 `virtio_blk_vq`，所以提交路径只要沿着这个指针取回真正的 `struct virtqueue`。

`virtblk_req` 是驱动为每个 request 准备的私有数据，通常放在 blk-mq request 后面的 PDU 区：

```text
request
  -> blk_mq_rq_to_pdu(rq)
  -> virtblk_req
       out_hdr
       status
       scatterlist / discard header 等
```

这就是 `tag_set.cmd_size` 的意义：blk-mq 给每个 request 额外分一段驱动私有空间，驱动不需要每次提交都单独分配核心命令对象。

### read / write 的方向

virtqueue 里要区分 out sg 和 in sg：

```text
out sg:
  guest -> device
  device 读取这些 buffer

in sg:
  device -> guest
  device 写入这些 buffer
```

所以：

| 请求 | out sg | in sg |
| --- | --- | --- |
| read | header | data buffer + status |
| write | header + data buffer | status |
| flush | header | status |
| discard | header + discard range descriptors | status |
| write zeroes | header + write-zeroes range descriptors | status |

这个方向很容易绕。判断方法是站在 device 视角：

```text
device 要读的 buffer  = out
device 要写的 buffer  = in
```

## notify / kick 做了什么

把 descriptor 放进 avail ring 之后，guest 要通知设备：

```text
virtqueue_kick()
  -> 根据 event idx / notification suppression 判断是否需要通知
  -> transport notify
  -> PCI notify register / MMIO notify / 其他 transport
  -> host 被唤醒处理队列
```

在 Linux virtio core 里，常见调用会拆成两步：

```text
virtqueue_kick_prepare(vq)
  -> 做必要的内存屏障，确保 descriptor / avail ring 先对 device 可见
  -> 根据 notification suppression / event idx 判断这次是否值得通知
  -> 返回 need_notify

virtqueue_notify(vq)
  -> 如果 need_notify 为真，调用 transport 注册的 notify 方法
  -> 对 virtio-pci 来说，通常是写 PCI notify BAR / queue notify 寄存器
  -> 对 virtio-mmio 来说，是写 QueueNotify MMIO 寄存器
  -> 对其他 transport，则走各自的通知机制
```

`virtqueue_notify()` 自己不搬运请求数据，也不把 `request` 传给 host。它更像按门铃：门铃里通常只有“哪个 virtqueue 有新东西”这类信息。host / device 被唤醒后，还是回到 guest 提前放好的 virtqueue 里读取 descriptor chain。

可以把提交顺序理解成：

```text
guest 写 descriptor table
  -> guest 写 avail ring
  -> guest 更新 avail idx
  -> virtqueue_kick_prepare() 保证发布顺序，并判断是否需要门铃
  -> virtqueue_notify() 写 transport doorbell
  -> device 开始消费 virtqueue
```

这里为什么要先 `kick_prepare()` 再 `notify()`？因为通知可能让另一个 CPU、host 线程、内核 vhost worker，甚至硬件立刻开始读 ring。通知之前必须保证 ring 内容已经按正确顺序发布出去。

不是每个请求都一定产生一次 VM exit 或 host 唤醒。virtio 有 event idx、interrupt suppression、batching 等机制，blk-mq 也有 plug 和批量派发。目标是让设备看到一批可处理请求，而不是每个 4 KiB I/O 都完整打一遍昂贵通知路径。

一个典型优化是：

```text
blk_start_plug()
  -> 多个 bio 合并 / 形成多个 request
  -> queue_rq 多次放入 virtqueue
  -> 最后统一 kick
```

具体是否合并 kick，取决于内核版本、virtqueue helper、blk-mq 批量接口和队列状态。

## host 侧发生什么

guest kick 之后，host 侧可能有不同后端：

| 后端 | 大致路径 |
| --- | --- |
| QEMU userspace virtio | QEMU 线程处理 virtqueue，访问 guest memory，再调用 host AIO / io_uring / 文件 I/O |
| vhost-blk / vhost-user | 把 virtqueue 处理放到 host 内核或专门用户态进程，减少 QEMU 主路径开销 |
| vDPA / 硬件 offload | 由硬件或专用设备直接处理 virtqueue |

从机制上看，host 要做：

```text
读取 avail ring
  -> 解析 descriptor chain
  -> 把 guest memory 地址转换成 host 可访问地址 / IOVA
  -> 根据 out_hdr.type 执行 read / write / flush / discard
  -> 写 data / status
  -> 填 used ring
  -> 触发 guest interrupt 或等待 guest polling
```

如果后端是 QEMU 文件：

```text
guest /dev/vda write
  -> host QEMU
  -> qcow2 / raw file
  -> host page cache 或 O_DIRECT
  -> host block layer
  -> host SSD / HDD / 网络盘
```

所以 virtio-blk 的性能不是只由 guest 驱动决定。host 文件格式、cache mode、AIO engine、底层存储和宿主机调度都会影响延迟。

## completion 路径

设备完成后，guest 可能通过中断或 polling 得知。

中断 completion 大致是：

```text
host 写 used ring
  -> 触发 guest interrupt
  -> virtio transport IRQ handler
  -> virtqueue callback
  -> virtblk_done(vq)
  -> virtqueue_get_buf()
  -> 找回 virtblk_req / request
  -> 读取 status byte
  -> 转成 blk_status_t
  -> blk_mq_end_request(rq, status)
```

然后 block layer 继续：

```text
blk_mq_end_request()
  -> 完成 request
  -> 完成 request 里的 bio
  -> bio_endio()
  -> 唤醒等待者
  -> 释放 tag
  -> 更新统计
```

completion 不一定在提交 request 的那个进程上下文里运行。它可能发生在：

- 硬中断上下文
- threaded IRQ
- softirq / block completion 路径
- polling 的调用上下文
- host / guest 配置相关的其他回调上下文

所以驱动 completion 路径必须短小，不能假设可以随便睡眠，也不能假设 `current` 是当初发起 I/O 的用户进程。

## status 到 block error 的转换

virtio status 是设备协议层状态，block layer 要的是 `blk_status_t`。

可以粗略理解成：

| virtio status | block layer 状态 |
| --- | --- |
| `VIRTIO_BLK_S_OK` | `BLK_STS_OK` |
| `VIRTIO_BLK_S_IOERR` | `BLK_STS_IOERR` |
| `VIRTIO_BLK_S_UNSUPP` | `BLK_STS_NOTSUPP` 或相近错误 |

上层最后看到的可能是：

- `read()` / `write()` 返回 `EIO`
- `fsync()` 返回延迟暴露的写回错误
- 文件系统把某个 page / folio 标记为 I/O error
- 块层统计里出现 failed I/O

错误不一定在最初的 buffered `write()` 上返回。对 page cache 写回来说，错误经常延迟到 writeback completion、`fsync()` 或后续相关操作。

## flush 和 write cache

普通 write completion 不一定等于持久化。virtio-blk 也有 write cache 和 flush 语义。

如果设备支持 flush：

```text
fsync()
  -> 文件系统写回数据 / 元数据
  -> block layer 下发 REQ_OP_FLUSH 或带 flush / FUA 语义的请求
  -> virtio-blk 转成 VIRTIO_BLK_T_FLUSH
  -> host backend flush 底层文件 / 块设备
  -> 完成后返回
```

重要区别：

| 动作 | 语义 |
| --- | --- |
| 普通 write | 数据提交给设备，可能只到 host page cache 或设备 volatile cache |
| flush | 要求之前的写推进到稳定存储语义对应的位置 |
| FUA | 要求这个写本身强制持久化，具体支持取决于设备和路径 |

虚拟化里还多一层风险：如果 host 后端、QEMU cache mode、底层存储或云盘没有正确兑现 flush，guest 里的 `fsync()` 语义也会被破坏。

常见 host cache mode 对语义和性能影响很大：

| host cache 模式 | 直观影响 |
| --- | --- |
| writeback | 性能好，但依赖正确 flush；普通写可能先到 host cache |
| writethrough | 更保守，普通写更慢 |
| none / direct | 绕过 host page cache，常用于数据库和虚拟磁盘 |
| unsafe | 可能忽略 flush，性能高但崩溃风险大 |

具体名字和行为以 hypervisor / QEMU 配置为准。分析数据可靠性时，不能只看 guest 是否调用了 `fsync()`。

## discard 和 write zeroes

virtio-blk 还可以支持非普通读写命令。

discard：

```text
guest fstrim / 文件系统释放空间
  -> block layer REQ_OP_DISCARD
  -> virtio-blk VIRTIO_BLK_T_DISCARD
  -> host 后端 punch hole / trim / unmap
```

write zeroes：

```text
guest 写零
  -> block layer REQ_OP_WRITE_ZEROES
  -> virtio-blk VIRTIO_BLK_T_WRITE_ZEROES
  -> host 后端高效置零 / punch hole / 底层 zero command
```

这些能力同样来自 feature negotiation。驱动会把最大范围、对齐、粒度设置给 block layer。设备不支持时，上层可能退化成普通写零，也可能直接返回不支持。

## 队列深度和 tag

blk-mq 用 tag 标识 in-flight request：

```text
request tag
  -> request
  -> driver private virtblk_req
  -> virtqueue descriptor chain
```

virtqueue 自己也有 queue size，也就是 descriptor 槽位数量。一个 request 可能消耗多个 descriptor：

```text
1 个 header descriptor
+ N 个 data descriptors
+ 1 个 status descriptor
```

所以最大并发 request 数不只受 blk-mq queue depth 影响，也受 virtqueue descriptor 数和每个 request 的 segment 数影响。

如果 virtqueue 暂时放不下新的 descriptor chain，驱动通常会向 block layer 返回资源不足：

```text
queue_rq()
  -> virtqueue_add_sgs() 失败 / ring full
  -> 返回 BLK_STS_RESOURCE 或停止队列
  -> block layer 稍后重试
```

这属于背压机制，不应该简单当成 I/O 失败。真正的 I/O 失败是设备完成后 status 表示错误，或者请求超时 / 设备重置等异常。

## 为什么需要内存屏障

virtqueue 是 guest 和 device 共享的内存数据结构。guest 填 descriptor、avail ring 后，device 可能在另一个 CPU、另一个进程、host 内核线程甚至硬件上读取。

所以顺序很重要：

```text
先写 descriptor 内容
再把 head 放进 avail ring
再更新 avail idx
最后 notify device
```

如果没有合适的内存屏障，device 可能先看到 avail idx 变了，却还看不到 descriptor 内容，读到半初始化数据。

virtio core helper 会处理这些屏障。驱动通常使用 `virtqueue_add_sgs()`、`virtqueue_kick()` 这类接口，而不是自己裸写 ring。

completion 方向也一样：

```text
device 先写 data / status
再写 used ring
再触发 interrupt

guest 先确认 used ring
再读取 data / status
```

这里的屏障是 virtio 正确性的核心，不只是性能细节。

## 中断、MSI-X 和 polling

virtio-pci 常见配置会使用 MSI-X。多队列时，通常可以让不同 virtqueue 使用不同 vector：

```text
virtqueue 0 -> MSI-X vector 0
virtqueue 1 -> MSI-X vector 1
virtqueue 2 -> MSI-X vector 2
...
```

这样 completion 可以分散到多个 CPU，降低单个中断向量压力。

但是中断也有成本。高 IOPS、小块 I/O 下，中断和 VM exit 可能成为瓶颈。常见优化方向包括：

- interrupt coalescing / event suppression
- blk-mq batching
- request merge
- 多队列
- polling
- vhost / vDPA 减少 QEMU userspace 路径开销

virtio-blk 是否支持、如何使用 polling，取决于内核版本、transport 和设备能力。读代码时可以关注 `blk_mq_ops.poll`、virtqueue polling helper、以及 block queue 的 polling 配置。

## request 超时和设备重置

block layer 会给 request 设置超时。若 request 长时间没有 completion，会进入 timeout 路径：

```text
request timeout
  -> blk-mq 调用驱动 timeout 回调
  -> 驱动判断设备是否还活着
  -> 可能 reset device / requeue request / fail request
```

virtio-blk 异常恢复通常要考虑：

- host 后端卡住
- virtqueue 中有 in-flight request
- device reset 后旧 descriptor 是否还能完成
- freeze / quiesce 队列，避免重置时继续提交
- 恢复后重新初始化 virtqueue
- 对未完成 request 返回错误或重新排队

这类路径比正常 I/O 更依赖具体内核版本。分析代码时要看当前版本的 freeze、remove、config change、reset 和 timeout 实现。

## 热插拔和配置变化

virtio 设备可能发生配置变化，比如容量变化。host 会触发 config changed 中断或回调，guest 驱动再读取配置。

大致路径：

```text
host 修改 virtio config
  -> 触发 config change
  -> guest virtio callback
  -> virtio-blk 读取新 capacity
  -> 更新 gendisk capacity
  -> 通知 block layer / userspace
```

用户态可能看到磁盘大小变化，需要重新扫描分区：

```text
blockdev --rereadpt /dev/vda
partprobe /dev/vda
```

热拔时要反过来：

```text
停止接收新 I/O
  -> quiesce / freeze queue
  -> 处理或失败 in-flight request
  -> del_gendisk()
  -> 删除 virtqueue
  -> 释放 tag_set 和驱动对象
```

驱动 remove 路径最怕还有 completion 回调访问已经释放的对象，所以必须先把队列和 virtqueue 停干净。

## 和 NVMe 的对比

virtio-blk 和 NVMe 都是 request-based blk-mq 驱动，但下层协议很不一样。

| 维度 | virtio-blk | NVMe |
| --- | --- | --- |
| 典型场景 | 虚拟机虚拟块设备 | 物理 / passthrough NVMe SSD |
| 提交结构 | virtqueue descriptor chain | NVMe submission queue entry |
| 完成结构 | used ring + status byte | completion queue entry |
| 数据描述 | scatterlist -> virtqueue descriptor | PRP / SGL |
| 后端 | host QEMU / vhost / hypervisor / vDPA | SSD controller |
| 地址语义 | virtio block sector | namespace LBA |
| 多队列 | `VIRTIO_BLK_F_MQ` | NVMe 原生多 SQ/CQ |
| 持久化 | 依赖 host 后端兑现 flush | 依赖控制器和介质兑现 flush / FUA |

共同点是：

```text
blk-mq request
  -> driver queue_rq()
  -> 构造设备命令
  -> 提交到某个硬件/虚拟队列
  -> completion
  -> blk_mq_end_request()
```

区别是 virtio-blk 多了一层 guest / host 边界，性能和语义强烈依赖虚拟化后端。

## 一次写请求完整串起来

以 guest 里普通 buffered write 的 writeback 为例：

```text
guest dirty page
  -> writeback worker
  -> 文件系统分配 / 查询 block mapping
  -> 构造 bio
  -> submit_bio()
  -> blk-mq 合并、调度、分配 request
  -> virtblk_queue_rq()
  -> 填 virtio_blk_outhdr(type = OUT, sector = ...)
  -> request pages 映射成 scatterlist
  -> header + data + status 组成 descriptor chain
  -> 放入 avail ring
  -> kick host
  -> host 后端读取 guest pages
  -> host 写底层存储
  -> host 写 status = OK
  -> host 放入 used ring
  -> guest interrupt
  -> virtblk_done()
  -> blk_mq_end_request()
  -> bio_endio()
  -> 文件系统 / page cache 清理 writeback 状态
```

如果是 `fsync()`，还会多出 flush / journal / metadata 同步路径：

```text
fsync()
  -> 等数据 writeback
  -> 提交文件系统日志或必要元数据
  -> block layer 下发 flush
  -> virtio-blk 发送 VIRTIO_BLK_T_FLUSH
  -> host 后端 flush
  -> completion 后 fsync 才能返回成功
```

## 常见误解

`/dev/vda` 不是 host 上真实存在的一块盘。它是 guest 看到的 virtio block device，host 后端可能是 raw 文件、qcow2 文件、LVM、Ceph RBD、NVMe 盘或别的东西。

virtio-blk 不负责文件系统语义。它不知道“这是哪个文件的第几个字节”，只知道 request 的 sector、长度、方向和 flags。

virtqueue descriptor 不是数据本身。descriptor 描述数据在哪里，真正的数据通常还在 page cache page 或 direct I/O buffer 对应的 guest memory 里。

普通 write completion 不等于掉电安全。持久化要看 flush / FUA、host cache mode、底层设备和云平台语义。

多队列不一定线性提升性能。如果瓶颈在 host 后端、单个 qcow2 锁、底层云盘限速、flush 频率或 CPU 调度，多 virtqueue 只能解决一部分问题。

virtio 是标准接口，不代表后端实现都一样。QEMU、vhost、vDPA、不同云厂商虚拟盘的性能和边界条件可能差很多。

## 排查性能时看哪里

guest 里可以看：

```text
lsblk -o NAME,ROTA,SIZE,MODEL
cat /sys/block/vda/queue/scheduler
cat /sys/block/vda/queue/max_sectors_kb
cat /sys/block/vda/queue/max_segments
cat /sys/block/vda/queue/write_cache
cat /sys/block/vda/stat
iostat -x 1
```

还可以关注：

| 指标 | 可能含义 |
| --- | --- |
| await 高 | I/O 延迟高，可能是 host 后端或底层盘慢 |
| util 高 | 设备或虚拟后端接近饱和 |
| avgqu-sz 高 | 队列堆积 |
| svctm 不可靠 | 现代 blk-mq / 虚拟设备下不要过度解读 |
| fsync 慢 | flush / journal / host cache / 底层介质慢 |
| CPU sys 高 | 小 I/O、virtqueue、中断、拷贝或 host/guest 切换成本高 |

host 侧要看：

- QEMU / vhost 线程 CPU
- 虚拟磁盘 cache mode
- raw 还是 qcow2
- host page cache / direct I/O
- host block layer 延迟
- 底层 SSD / 云盘限速
- cgroup / throttling
- NUMA 和 vCPU / iothread 绑定

## 读代码时的抓手

读 Linux virtio-blk 代码时，可以按这几条线找：

| 线索 | 看什么 |
| --- | --- |
| probe | 设备发现、feature negotiation、读取 capacity、初始化 tag_set / disk |
| queue setup | virtqueue 数量、MSI-X vector、blk-mq `nr_hw_queues` |
| queue_rq | request 到 header / sg / descriptor chain 的转换 |
| completion callback | used ring 到 `blk_mq_end_request()` |
| config changed | 容量变化、设备配置更新 |
| remove / freeze | 热拔、挂起、恢复、队列停止 |
| timeout | request 卡住后的恢复策略 |
| queue limits | max segments、block size、discard、write zeroes、flush |

核心函数名会随内核版本略有变化，但主线一般逃不开：

```text
probe / init
  -> find virtqueues
  -> init blk-mq tag_set
  -> alloc disk
  -> set capacity / limits
  -> add disk

queue_rq
  -> build virtblk_req
  -> map request to sg
  -> add sgs to virtqueue
  -> kick

done
  -> get used buffers
  -> read status
  -> end request
```

## 一段能讲清楚的话

virtio-blk 是 guest Linux 里的 virtio 块设备驱动。它对上接 Linux block layer，用 blk-mq 接收 request；对下接 virtqueue，把 request 编码成 `virtio_blk_outhdr + data sg + status byte` 的 descriptor chain。提交时，guest 把 descriptor chain 放进 avail ring 并 kick host；host 后端读取 descriptor，访问 guest memory，完成真实 I/O 后写 status、填 used ring、触发 guest completion。guest 驱动在 completion 回调里取回 request，把 virtio status 转成 block layer 状态，并调用 `blk_mq_end_request()`，于是 bio、page cache、文件系统等待者继续被唤醒。

所以分析 virtio-blk 时要同时看三层：guest block layer 是否给出了合适的 request，virtio-blk 是否正确组织 virtqueue 和 completion，host 后端是否真正高效且正确地兑现 read / write / flush 语义。
