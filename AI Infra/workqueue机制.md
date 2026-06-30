# Linux workqueue 机制

先记住一个核心点：

workqueue 是 Linux 内核里把工作延后到进程上下文执行的机制。它常用来把中断、定时器、软中断或其他不能睡眠的路径里的事情，丢给内核线程 `kworker` 稍后执行。

粗略路径是：

```text
内核某处
  -> 初始化 work_struct
  -> queue_work() / schedule_work()
  -> work 挂到某个 worker_pool
  -> 唤醒或创建 kworker
  -> kworker 取出 work
  -> 调用 work->func(work)
  -> work 完成，更新状态，继续处理下一个 work
```

和中断、softirq、tasklet 最大的区别是：

workqueue 的回调运行在普通内核线程上下文里，所以可以睡眠、可以拿 mutex、可以做可能阻塞的内存分配，也可以发起同步 I/O。

这也是它存在的主要意义。

这里说的是普通 kworker workqueue。新内核里还有 `WQ_BH` 这种 BH workqueue，它更像 softirq 的便利封装，work item 在 softirq 上下文执行，不能睡眠。日常说 workqueue 能睡眠，默认指的不是 `WQ_BH`。

## 为什么需要 workqueue

很多内核路径不能直接做重活。

比如中断处理函数里，应该尽快确认硬件事件、取走必要状态，然后返回。它不能长时间占 CPU，也不能睡眠。如果在中断里直接做复杂处理，就会拖慢整个系统的中断响应。

又比如 timer、softirq 这类上下文，也不适合做可能阻塞的事情。

workqueue 的思路是：

```text
快路径：
  记录事件
  把真正的工作排队
  立刻返回

慢路径：
  由 kworker 在线程上下文里慢慢处理
```

典型用途：

| 场景 | 为什么适合 workqueue |
| --- | --- |
| 中断下半部 | top half 只做最小处理，剩下的放到 worker |
| 设备驱动异步任务 | 复位设备、重新配置硬件、处理复杂状态机 |
| 后台清理 | 释放资源、回收缓存、批量整理状态 |
| 网络 / 存储辅助任务 | 不适合在中断或 softirq 里完成的阻塞工作 |
| 内核模块异步初始化 / 销毁 | 避免在调用路径里同步等待全部工作完成 |
| writeback、内存管理等后台任务 | 周期性或按需把工作交给 kworker 执行 |

一句话：只要内核里有“现在不能做，或者不想在当前上下文里做，但稍后必须做”的事情，就可能用 workqueue。

## 它不是简单创建线程

早期理解 workqueue 时，很容易把它想成：

```text
每个 workqueue 自己养一组线程
```

现代 Linux 不是这个模型。现代实现叫 **CMWQ**，也就是 concurrency-managed workqueue。

它的核心思想是：

```text
workqueue_struct 是用户看到的队列接口
worker_pool 是实际执行 work 的线程池
kworker 是 worker_pool 里的执行线程
pool_workqueue 把一个 workqueue_struct 和一个 worker_pool 连接起来
```

也就是说，一个 workqueue 不一定直接拥有一批固定线程。很多 workqueue 可以共享底层 worker pool。内核统一管理 worker 数量、并发度、CPU 绑定、空闲 worker、按需创建 worker 等事情。

大致关系：

```text
workqueue_struct
  -> pool_workqueue on CPU0
       -> worker_pool CPU0
          -> kworker/0:0
          -> kworker/0:1
  -> pool_workqueue on CPU1
       -> worker_pool CPU1
          -> kworker/1:0
          -> kworker/1:1

unbound workqueue
  -> pool_workqueue
       -> unbound worker_pool
          -> kworker/u...
```

用户把 `work_struct` 排到某个 `workqueue_struct`。真正执行时，这个 work 会被放进对应的 `pool_workqueue`，再进入某个 `worker_pool` 的工作链表，最后由 `kworker` 线程取出来执行。

## 最小使用方式

最简单的内核代码大概是这样：

```c
static void my_workfn(struct work_struct *work)
{
        /* 这里是进程上下文，可以睡眠 */
}

static DECLARE_WORK(my_work, my_workfn);

void some_event_handler(void)
{
        schedule_work(&my_work);
}
```

`schedule_work()` 使用的是系统默认 workqueue，也就是 `system_wq`。

如果需要自己创建队列：

```c
static struct workqueue_struct *my_wq;
static struct work_struct my_work;

static void my_workfn(struct work_struct *work)
{
        /* do something */
}

int init(void)
{
        my_wq = alloc_workqueue("my_wq", WQ_UNBOUND, 0);
        INIT_WORK(&my_work, my_workfn);
        queue_work(my_wq, &my_work);
        return 0;
}

void exit(void)
{
        cancel_work_sync(&my_work);
        destroy_workqueue(my_wq);
}
```

几个常见 API：

| API | 作用 |
| --- | --- |
| `INIT_WORK(work, func)` | 初始化一个普通 work |
| `DECLARE_WORK(name, func)` | 静态定义并初始化 work |
| `queue_work(wq, work)` | 把 work 排到指定 workqueue |
| `schedule_work(work)` | 把 work 排到默认 `system_wq` |
| `queue_work_on(cpu, wq, work)` | 尽量把 work 放到指定 CPU 对应的 pool |
| `flush_work(work)` | 等这个 work 如果已经排队或正在执行，则执行完成 |
| `flush_workqueue(wq)` | 等某个 workqueue 上当前一批 work 完成 |
| `cancel_work_sync(work)` | 取消未执行的 work；如果正在执行则等待它结束 |
| `INIT_DELAYED_WORK(dwork, func)` | 初始化延迟 work |
| `queue_delayed_work(wq, dwork, delay)` | 延迟一段 jiffies 后排队执行 |
| `mod_delayed_work(wq, dwork, delay)` | 新增或修改 delayed work 的触发时间 |

## 普通 work 和 delayed work

普通 `work_struct` 是“尽快执行”。

`delayed_work` 是“过一段时间再执行”。它本质上是：

```text
delayed_work
  -> timer
  -> work_struct
```

延迟时间没到之前，它主要挂在 timer 机制里。timer 到期后，内核才把里面的 `work_struct` 真正 queue 到 workqueue。

路径大概是：

```text
queue_delayed_work()
  -> 设置 timer 到期时间
  -> timer 到期
  -> delayed_work_timer_fn()
  -> queue_work()
  -> kworker 执行 work function
```

所以 delayed work 的回调也运行在 kworker 线程上下文里，不是在 timer 中断上下文里执行。

## 关键数据结构

不同内核版本字段会有变化，但核心对象关系很稳定。

## `struct work_struct`

`work_struct` 表示一个具体任务。

它大概包含：

```c
struct work_struct {
        atomic_long_t data;
        struct list_head entry;
        work_func_t func;
};
```

可以这么理解：

| 字段 | 含义 |
| --- | --- |
| `data` | 编码 work 的状态，比如 pending 标记、所属 pool / pwq 信息 |
| `entry` | 挂到 worker_pool 工作链表里的链表节点 |
| `func` | 真正要执行的回调函数 |

`work_struct` 通常嵌在驱动或内核模块自己的对象里：

```c
struct my_device {
        struct work_struct reset_work;
        struct mutex lock;
        int state;
};
```

回调里再用 `container_of()` 找回外层对象：

```c
static void reset_workfn(struct work_struct *work)
{
        struct my_device *dev =
                container_of(work, struct my_device, reset_work);

        mutex_lock(&dev->lock);
        /* reset device */
        mutex_unlock(&dev->lock);
}
```

这也是内核常见写法：`work_struct` 不携带用户参数，参数由外层对象承载。

## `struct workqueue_struct`

`workqueue_struct` 是逻辑上的 workqueue。

使用者看到的是它：

```c
struct workqueue_struct *wq;

wq = alloc_workqueue("name", flags, max_active);
queue_work(wq, work);
```

它记录的信息包括：

| 内容 | 作用 |
| --- | --- |
| 名字 | 影响 `kworker` / debug 信息，方便排查 |
| flags | 是否 unbound、highpri、freezable、mem reclaim 等 |
| max_active | 限制每个 CPU 或 pool 上同时活跃的 work 数 |
| pwq 列表 | 连接到各个底层 worker_pool |
| flush 状态 | 支持 `flush_workqueue()` 等同步操作 |
| rescuer | `WQ_MEM_RECLAIM` 队列可能有专用救援线程 |

重点是：它不是简单的“线程数组”。它更像一个逻辑队列和策略对象。

## `struct worker_pool`

`worker_pool` 是真正管理 worker 线程的地方。

它大概负责：

| 内容 | 作用 |
| --- | --- |
| `worklist` | 等待执行的 work 链表 |
| `workers` | 属于这个 pool 的 worker 列表 |
| `idle_list` | 当前空闲的 worker |
| `nr_workers` / `nr_idle` | worker 数量统计 |
| `lock` | 保护 pool 内部状态 |
| `cpu` / `attrs` | bound pool 绑定 CPU；unbound pool 使用属性描述 |
| manager 逻辑 | 按需创建、唤醒、回收 worker |

一个容易混淆的点：普通驱动或子系统通常不会自己创建 `worker_pool`。你创建的是 `workqueue_struct`：

```c
alloc_workqueue("my_wq", flags, max_active);
```

`worker_pool` 是 workqueue core 的内部资源，由内核根据 workqueue 的属性自动创建、复用和管理。调用者通过 `flags`、`max_active`、unbound attrs 等参数影响映射到哪类 pool，但不是直接 `alloc_worker_pool()`。

内核里主要有这些 pool：

| pool 类型 | 典型 worker 名字 | 执行上下文 | 说明 |
| --- | --- | --- | --- |
| per-CPU normal pool | `kworker/0:1`、`kworker/3:2` | kworker 线程 | 绑定某个 CPU 的普通 worker pool |
| per-CPU highpri pool | `kworker/0:1H`、`kworker/3:0H` | kworker 线程 | 绑定某个 CPU 的高优先级 worker pool，`H` 表示 high priority |
| unbound normal pool | `kworker/u8:0`、`kworker/u16:2` | kworker 线程 | 不绑定提交 work 的 CPU，由调度器在允许 CPU 集合内调度 |
| unbound highpri pool | `kworker/u8:1H` | kworker 线程 | unbound 的高优先级 worker pool |
| BH pseudo pool | 通常不表现为普通 `kworker/...` | softirq 上下文 | `WQ_BH` 使用，不是真正睡眠型 kworker 线程池 |

这里的 worker 名字不是稳定 ABI，只适合观察和调试。

命名大致可以这么读：

| 名字片段 | 含义 |
| --- | --- |
| `kworker/0:1` | CPU0 上编号为 1 的 bound worker |
| `kworker/2:0` | CPU2 上编号为 0 的 bound worker |
| `kworker/2:0H` | CPU2 上的 highpri bound worker |
| `kworker/u8:2` | unbound worker；`u` 表示 unbound，`8` 通常是 unbound pool id，`2` 是 worker id |
| `kworker/u16:1H` | highpri unbound worker |

不要把业务逻辑依赖在线程名上。不同内核版本、配置、CPU 拓扑和 workqueue 属性都可能让名字看起来不一样。

bound workqueue 通常使用 per-CPU pool。也就是每个 CPU 有自己的 normal / highpri worker pool。

unbound workqueue 不绑定具体 CPU，worker 可以在一组 CPU 上调度，适合不要求 CPU locality、可能阻塞较久或希望跨 CPU 均衡的任务。

## `struct pool_workqueue`

`pool_workqueue` 经常缩写成 `pwq`。

它连接：

```text
workqueue_struct <-> worker_pool
```

为什么需要它？

因为一个逻辑 workqueue 可能映射到多个 pool。比如 bound workqueue 在每个 CPU 上都有一个对应 pool；unbound workqueue 也可能因为 NUMA、cpumask、priority 属性映射到不同 pool。

`pwq` 里会记录：

| 内容 | 作用 |
| --- | --- |
| 所属 `workqueue_struct` | 这是哪个逻辑队列 |
| 所属 `worker_pool` | 实际由哪个 pool 执行 |
| `nr_active` | 当前活跃 work 数 |
| `max_active` | 并发上限 |
| `inactive_works` | 超过并发上限后暂存的 work |
| 引用计数和 flush 颜色 | 支持生命周期和 flush 同步 |

`max_active` 是通过 `pwq` 发挥作用的。它决定同一个 workqueue 在某个 pool 上最多同时激活多少 work。

## `struct worker`

`worker` 表示一个具体的 kworker 线程。

它大概记录：

| 内容 | 作用 |
| --- | --- |
| `task` | 对应的 `task_struct`，也就是内核线程 |
| `pool` | 它属于哪个 worker_pool |
| `current_work` | 当前正在执行的 work |
| `current_func` | 当前正在执行的函数 |
| `current_pwq` | 当前 work 来自哪个 pwq |
| flags | idle、running、manager 等状态 |

执行 work 的最终动作就是 worker 线程调用：

```c
work->func(work);
```

## bound 和 unbound workqueue

workqueue 有两种非常重要的执行模型。

## bound workqueue

bound workqueue 和 CPU 绑定。

如果在 CPU0 上 queue 一个 bound work，它通常会被放到 CPU0 对应的 pool，由 CPU0 上的 kworker 执行。

优点：

- cache locality 更好
- per-CPU 数据访问更自然
- 行为更可预测

缺点：

- 如果某个 CPU 上 work 很多，其他 CPU 不一定能直接帮忙
- 长时间阻塞的 work 可能影响这个 CPU pool 的调度

默认 `alloc_workqueue()` 如果不带 `WQ_UNBOUND`，就是 bound 风格。

## unbound workqueue

unbound workqueue 不固定绑在 queue work 的那个 CPU 上。

它适合：

- 可能阻塞较久的任务
- 不关心 CPU locality 的任务
- 希望由调度器在允许的 CPU 集合里均衡的任务
- 后台清理、设备管理这类不适合卡住某个 CPU pool 的工作

常用创建方式：

```c
alloc_workqueue("name", WQ_UNBOUND, max_active);
```

`kworker/u...` 通常就是 unbound worker。

## 常见 workqueue flags

创建 workqueue 时可以传 flags：

```c
alloc_workqueue("name", flags, max_active);
```

常见 flags：

| flag | 含义 |
| --- | --- |
| `WQ_PERCPU` | per-CPU / bound workqueue，强调 CPU locality |
| `WQ_UNBOUND` | 不绑定提交 work 的 CPU |
| `WQ_BH` | BH workqueue，在 softirq 上下文执行，不能睡眠 |
| `WQ_HIGHPRI` | 使用高优先级 worker pool |
| `WQ_MEM_RECLAIM` | 这个队列参与内存回收路径，需要 rescuer 避免死锁 |
| `WQ_FREEZABLE` | 系统 suspend / hibernate 时可被冻结 |
| `WQ_CPU_INTENSIVE` | CPU 密集型 work，不应占用普通并发管理额度 |
| `WQ_SYSFS` | 允许通过 sysfs 暴露和调整部分属性 |
| `WQ_POWER_EFFICIENT` | 在开启节能策略时倾向省电调度 |

如果需要严格有序执行，通常用 `alloc_ordered_workqueue()` 创建 ordered workqueue，而不是把它当成普通 flag 随手塞给 `alloc_workqueue()`。

几个容易误解的点：

`WQ_HIGHPRI` 不是实时调度。它只是使用 highpri worker pool，优先级高一些，不等于可以无限制抢占系统。

`WQ_CPU_INTENSIVE` 不是让 work 跑得更快。它是告诉 workqueue 并发管理逻辑：这个 work 主要吃 CPU，不要因为它正在运行就阻止 pool 继续启动其他普通 work。

`WQ_CPU_INTENSIVE` 主要对 bound workqueue 有意义；对 unbound workqueue 基本没有意义，因为 unbound 本来就更依赖调度器做并发管理。

`WQ_MEM_RECLAIM` 很重要。凡是可能在内存回收路径中被依赖的 workqueue，都应该考虑它。否则系统低内存时，worker 创建或内存分配失败，可能导致“需要 worker 来释放内存，但没有内存创建 worker”的死锁。

## 系统默认 workqueue

内核提供了一些全局 workqueue，很多普通场景不用自己创建。

常见的有：

| 名称 | 用途 |
| --- | --- |
| `system_wq` | 默认普通队列，`schedule_work()` 用它 |
| `system_highpri_wq` | 高优先级普通系统队列 |
| `system_long_wq` | 适合可能运行较久的 work |
| `system_unbound_wq` | 默认 unbound 系统队列 |
| `system_freezable_wq` | suspend 时可冻结 |
| `system_power_efficient_wq` | 省电倾向的系统队列 |

小型、短任务可以用系统队列。

如果 work 有特殊需求，最好自己创建 workqueue：

- 需要严格并发上限
- 需要 `WQ_MEM_RECLAIM`
- 需要 ordered 语义
- 任务可能长期阻塞
- 不希望和其他系统 work 互相影响

## 任务执行的完整生命周期

下面按一次普通 `queue_work()` 看完整路径。

## 1. 初始化 work

通常在设备对象、模块对象或文件系统对象初始化时：

```c
INIT_WORK(&obj->work, obj_workfn);
```

这一步会设置回调函数，并把内部状态初始化成“没有 pending”。

注意：同一个 `work_struct` 在重新初始化前，不能还处在 pending 或 running 状态。否则状态会被破坏。

## 2. 提交 work

事件发生时调用：

```c
queue_work(wq, &obj->work);
```

内部大致做几件事：

```text
检查并设置 WORK_STRUCT_PENDING
  -> 如果已经 pending，返回 false
  -> 根据 wq 类型选择目标 pwq / worker_pool
  -> 把 work 插入队列
  -> 必要时唤醒 idle worker 或创建新 worker
  -> 返回 true
```

`queue_work()` 的返回值很有用：

| 返回值 | 含义 |
| --- | --- |
| `true` | 这次成功把 work 加入队列 |
| `false` | work 已经在队列里，没有重复加入 |

这意味着同一个 `work_struct` 不能靠反复 `queue_work()` 堆出多个实例。它不像用户态任务队列那样每次 push 一个新任务对象。一个 `work_struct` 同一时间最多表示一个 pending 实例。

如果需要多个独立任务，要分配多个 `work_struct`，或者用自己的链表 / 队列在一个 work 回调里批量消费。

## 3. 选择 pool

对 bound workqueue：

```text
当前 CPU 或指定 CPU
  -> 该 CPU 对应的 worker_pool
  -> 对应 pool_workqueue
```

对 unbound workqueue：

```text
workqueue attributes
  -> cpumask / NUMA / priority 等
  -> 找到或复用 unbound worker_pool
  -> 对应 pool_workqueue
```

这个选择过程决定 work 最终由哪类 `kworker` 执行。

## 4. 进入 active 或 inactive

workqueue 需要控制并发度。

如果当前 `pwq->nr_active < max_active`，work 可以直接进入 active 状态，挂到 pool 的 `worklist`，等待 worker 执行。

如果已经达到并发上限，work 会先进入 `inactive_works`：

```text
queue_work()
  -> pwq active 数未满
       -> 放入 pool->worklist
  -> pwq active 数已满
       -> 放入 pwq->inactive_works
```

当某个 active work 完成后，workqueue 会把 inactive 里的后续 work 激活。

这就是 `max_active` 的基本含义：限制同一个 workqueue 在一个 pool 上同时“活跃”的 work 数量。

更准确地说，`max_active` 是 per-CPU 属性，即使对 unbound workqueue 也是这样理解。传 `0` 表示使用内核默认值，普通场景通常推荐这么做，除非你明确需要限流。如果要严格“一次只跑一个 work 并按提交顺序执行”，不要靠 `max_active = 1` 拼出来，而应该用 `alloc_ordered_workqueue()`。

## 5. 唤醒或创建 worker

如果 pool 里有空闲 worker，就唤醒一个。

如果没有空闲 worker，而且 workqueue 判断当前需要更多并发，就可能创建新的 kworker。

这里有一个重要目标：

```text
既不能 worker 太少，导致 work 堆积；
也不能 worker 太多，导致线程爆炸和调度开销过高。
```

CMWQ 的“concurrency-managed”主要就在这里：它会根据 worker 是否阻塞、pool 是否还有待执行 work、当前是否已有 running worker 等状态，动态维持合适并发度。

如果一个 worker 执行 work 时睡眠了，pool 可能会启动另一个 worker 继续处理队列，避免整个 pool 因一个阻塞任务停住。

## 6. kworker 执行 work

worker 线程主循环大概是：

```text
while (true):
  如果没有 work:
    进入 idle / sleep
  从 pool->worklist 取一个 work
  标记 current_work/current_func/current_pwq
  清除 work 的 pending 状态
  释放 pool lock
  调用 work->func(work)
  重新拿 pool lock
  更新统计和状态
  激活 inactive work
```

真正执行函数时，已经不持有 pool 的自旋锁。否则 work 回调里稍微做点复杂事情就会把整个 pool 锁死。

`pending` 标记通常会在回调执行前清除。这带来一个行为：

如果 work 正在执行，其他路径再次 `queue_work()`，可能会让它在当前执行结束后再跑一轮。

所以 work 回调要写成能够处理重复触发、合并事件或重新检查状态的形式。不要假设“一次 queue 必然对应一个外部事件，也只处理那一个事件”。

## 7. work 回调里做事

回调运行在 kworker 线程上下文中。

它可以：

- 睡眠
- 拿 mutex
- 调用可能阻塞的函数
- 分配普通内存
- 发起同步 I/O
- 访问当前进程无关的内核对象

它不应该：

- 长时间占 CPU 不让出
- 无限制循环
- 依赖用户进程的地址空间
- 假设 `current` 是提交 work 的那个进程
- 在没有生命周期保护的情况下访问可能被释放的对象

`current` 是 kworker，不是原来的用户进程或中断发生时被打断的进程。

这点很重要。workqueue 只是延后执行内核函数，不会保存和恢复提交者的用户态上下文。

## 8. 完成和后续激活

回调返回后，worker 会：

```text
清理 current_work/current_func/current_pwq
减少 pwq 的 in-flight / active 计数
如果 inactive_works 里还有任务，激活后续 work
检查是否需要继续处理 pool->worklist
必要时进入 idle
```

如果有人正在 `flush_work()` 或 `flush_workqueue()` 等待，完成路径也会唤醒等待者。

## flush 和 cancel

workqueue 的同步 API 是使用时最容易出 bug 的部分。

## `flush_work(work)`

`flush_work()` 的语义是：

如果这个 work 在调用 `flush_work()` 时已经排队或正在执行，那就等它执行完成。

它不是“永远保证这个 work 以后不会再执行”。如果其他 CPU 在 flush 之后又 queue 了它，它仍然会执行。

## `flush_workqueue(wq)`

`flush_workqueue()` 等待某个 workqueue 上当前已经进入队列的一批 work 完成。

它需要处理一个问题：flush 的时候，work 之间可能互相 queue 新 work。

内核通常用类似“颜色”的机制给 work 分批：

```text
flush 开始时，标记当前 batch
等待这个 batch 里的 work 完成
后续新排入的 work 属于新 batch，不一定被这次 flush 等待
```

不要把 `flush_workqueue()` 理解成“让整个队列永远变空并保持为空”。它只是一个同步点。

## `cancel_work_sync(work)`

`cancel_work_sync()` 做两件事：

```text
如果 work 还没开始执行，就从队列里删掉
如果 work 正在执行，就等待它执行完
```

返回后，这个 work 不会还在当前那次执行中。

但同样要注意：如果别的路径在 `cancel_work_sync()` 返回后又 queue 它，它还是会再执行。所以对象销毁时，通常要先阻止新的 queue 来源，再 cancel / flush。

常见销毁顺序：

```text
标记对象正在销毁，阻止新事件继续 queue work
  -> 关闭硬件中断 / timer / callback
  -> cancel_work_sync()
  -> 释放对象内存
```

如果先释放对象，再 cancel work，回调里 `container_of()` 拿到的外层对象就可能已经是悬空指针。

## delayed work 的取消

delayed work 需要区分两个阶段：

```text
timer 还没到期
timer 到期后 work 已经入队或执行
```

常用 API：

| API | 作用 |
| --- | --- |
| `cancel_delayed_work()` | 尝试取消 delayed work，但不等待正在执行的回调 |
| `cancel_delayed_work_sync()` | 取消并等待可能正在执行的回调结束 |
| `flush_delayed_work()` | 如果 delayed work 还没入队，会推动它入队并等待执行完成 |

模块退出、设备 detach、对象释放时，通常用 `_sync` 版本。

## 内存回收和 rescuer

`WQ_MEM_RECLAIM` 值得单独记。

假设某个文件系统或驱动在内存回收路径里必须依赖一个 work 完成，才能释放更多内存。如果系统已经极度缺内存，而 workqueue 又需要分配内存来创建 worker，就可能出现死锁：

```text
需要 worker 执行 work 才能释放内存
  -> 创建 worker 又需要内存
  -> 内存不足，创建不了 worker
  -> 回收路径卡住
```

带 `WQ_MEM_RECLAIM` 的 workqueue 会配一个 rescuer 线程。正常 worker 不够或创建失败时，rescuer 可以接管这些 work，保证内存回收路径有前进能力。

所以如果一个 workqueue 参与 reclaim 路径，或者它的完成是 reclaim 能继续推进的前提，就应该使用 `WQ_MEM_RECLAIM`。

这不是性能优化，而是死锁规避。

## ordered workqueue

有些任务必须严格按提交顺序执行，比如某些状态机、设备命令序列、日志提交序列。

这时可以用 ordered workqueue：

```c
alloc_ordered_workqueue("name", flags);
```

它的效果接近：

```text
同一队列上一次只执行一个 work
后提交的 work 等前面的 work 完成
```

代价是并发度低。能用锁或状态机解决的，不一定都要 ordered；但如果顺序本身就是语义的一部分，ordered workqueue 很直接。

## workqueue 和其他下半部机制对比

| 机制 | 上下文 | 能否睡眠 | 典型用途 |
| --- | --- | --- | --- |
| hardirq | 中断上下文 | 不能 | 最快确认硬件事件 |
| softirq | 软中断上下文 | 不能 | 网络收包、定时器等高频事件 |
| tasklet | 软中断上下文 | 不能 | 旧式下半部，现代代码里逐渐少用 |
| threaded irq | 专用 IRQ 线程 | 可以 | 把中断处理线程化 |
| workqueue | kworker 线程上下文 | 可以 | 通用异步后台工作 |
| kthread | 自己创建的内核线程 | 可以 | 长期存在、逻辑复杂、自主管理循环的任务 |

workqueue 和 kthread 的区别：

workqueue 更适合“有任务就执行，没任务就睡眠”的异步工作。线程池、并发管理、CPU hotplug、worker 生命周期由内核统一处理。

kthread 更适合你确实需要一个长期运行的线程，有自己的主循环、等待条件、退出协议和调度策略。

## 并发和重入

同一个 `work_struct` 不会因为一次 `queue_work()` 被放进队列多份。

同一个 `work_struct` 也不会被两个 worker 同时执行。即使它在正在运行时被再次 queue，workqueue 也会把它处理成后续再跑一轮，而不是让同一个 work item 重入执行。

但是要注意几种并发：

1. 同一个 work function 可以被多个不同的 `work_struct` 同时执行。
2. 一个 work 正在执行时，可能被再次 queue，导致结束后再执行一轮。
3. unbound workqueue 上，不同 work 可能在不同 CPU 上同时执行。
4. 即使是 bound workqueue，不同 CPU 上的 per-CPU pool 也可能并行执行同一个 workqueue 的不同 work。

所以共享状态仍然需要锁、atomic、refcount、RCU 或其他生命周期保护。

workqueue 不等于自动串行化。只有 ordered workqueue，或者 `max_active = 1` 且映射关系符合预期时，才有更强的串行效果。

## CPU hotplug

bound workqueue 和 CPU 绑定，所以 CPU online / offline 时，workqueue 也要迁移和重建对应关系。

大概会发生：

```text
CPU offline
  -> 对应 bound worker_pool 不再正常接收执行
  -> 未完成 work 迁移到合适 CPU / pool
  -> worker 线程 park 或退出

CPU online
  -> 重新准备 per-CPU worker_pool
  -> 恢复对应 kworker
```

这部分由 workqueue 核心处理。普通使用者不需要自己管理 CPU 热插拔，但写 per-CPU 数据时要知道：work 是否真的只能在某个 CPU 上执行，取决于你使用的 queue API 和 workqueue 类型。

## 调试和观察

系统里看到大量 `kworker` 不一定是问题。workqueue 本来就是很多内核后台任务的公共执行层。

常见观察方式：

```bash
ps -eLo pid,tid,comm,wchan | grep kworker
cat /proc/<pid>/stack
cat /sys/kernel/debug/workqueue/*
```

如果开启 ftrace / tracepoint，可以看 workqueue 事件：

```text
workqueue:workqueue_queue_work
workqueue:workqueue_activate_work
workqueue:workqueue_execute_start
workqueue:workqueue_execute_end
```

分析 kworker CPU 高时，重点不是“哪个 kworker 高”，而是要找：

```text
这个 kworker 正在执行哪个 work function
这个 function 属于哪个模块 / 子系统
为什么它被频繁 queue
回调里是否阻塞、忙等或重复重排队
```

`/proc/<pid>/stack`、ftrace、perf、lockdep、hung task 日志都可能有帮助。

## 常见坑

## 对象生命周期错误

典型 bug：

```text
对象里嵌了 work_struct
  -> queue_work()
  -> 对象被释放
  -> kworker 后来执行 work
  -> container_of() 得到已释放对象
```

解决思路：

- 对象释放前阻止新的 queue 来源
- 使用 `cancel_work_sync()` 或 `flush_work()`
- 必要时用 refcount 保证回调执行期间对象还活着

## 在 work 里无条件重排自己

比如：

```c
static void poll_workfn(struct work_struct *work)
{
        do_poll();
        queue_work(wq, work);
}
```

这会变成一个永不停止的后台循环。如果没有退出条件、延迟、限速或状态检查，可能把 CPU 打满。

周期性任务通常用 delayed work：

```c
queue_delayed_work(wq, &dwork, msecs_to_jiffies(1000));
```

## 忘记 `_sync` 取消

模块退出或设备移除时，只调用 `cancel_work()` 可能不够。因为它不等待已经开始执行的回调。

销毁对象时更常见的是：

```c
cancel_work_sync(&obj->work);
cancel_delayed_work_sync(&obj->dwork);
```

## 在 flush 等待自己

如果 work 回调内部对同一个 work 或同一个队列做不合适的 flush，可能死锁。

比如一个 work 正在执行，却等待“包含自己在内的这批 work 完成”，那就永远等不到。

所以 flush / cancel API 要看调用上下文：是否可能从目标 work 自己的回调里调用，是否持有回调也需要的锁。

## 锁顺序问题

常见死锁形状：

```text
线程 A:
  mutex_lock(dev->lock)
  flush_work(&dev->work)

kworker:
  dev_workfn()
    mutex_lock(dev->lock)
```

线程 A 拿着 `dev->lock` 等 work 完成，而 work 正在等 `dev->lock`，就死锁了。

flush 前不要持有 work 回调必需的锁，除非你非常确定回调不会走到那条路径。

## 把 workqueue 当实时机制

workqueue 是异步执行机制，不是实时调度保证。

work 什么时候执行，受这些因素影响：

- 当前 pool 是否有空闲 worker
- `max_active` 是否达到上限
- CPU 调度压力
- worker 是否被其他 work 阻塞
- workqueue 类型和 flags
- 系统 suspend / freezer 状态

如果需要严格延迟上界，普通 workqueue 不是足够强的抽象。

## 一段能讲清楚的话

Linux workqueue 是内核通用的异步执行机制。调用者把一个 `work_struct` 提交到 `workqueue_struct`，workqueue 核心根据队列属性选择对应的 `pool_workqueue` 和 `worker_pool`，把 work 挂入等待链表，并唤醒或创建 `kworker`。`kworker` 在线程上下文中取出 work，清除 pending 状态，调用 `work->func(work)`，回调返回后更新 active 计数、唤醒 flush 等待者，并继续处理后续 work。

它解决的是“当前上下文不能睡眠或不适合做重活”的问题。中断、timer、softirq 等路径可以只做最小处理，把复杂、可能阻塞的工作延后给 kworker。现代 CMWQ 不再是每个队列固定养一组线程，而是让多个逻辑 workqueue 共享底层 worker pool，由内核统一管理并发度、CPU 绑定、worker 创建和回收。

使用时最重要的是三件事：第一，搞清楚 work 回调运行在 kworker 上，不是提交者上下文；第二，保护好对象生命周期，释放前阻止新 queue 并 `cancel_work_sync()`；第三，理解并发语义，workqueue 不是天然串行化机制，除非显式使用 ordered 队列或设计了自己的同步。
