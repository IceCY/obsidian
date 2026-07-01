# Linux softirq 机制

先记住一个核心点：

softirq 是 Linux 内核里的高优先级“下半部”机制。它把硬中断里不适合做完的工作延后执行，但仍然运行在原子上下文里，不能睡眠。

粗略路径是：

```text
硬件中断
  -> hardirq/top half 做最小处理
  -> raise_softirq() 标记某类 softirq pending
  -> 硬中断返回 / local_bh_enable() / ksoftirqd
  -> __do_softirq() 扫描 pending 位图
  -> 调用对应 softirq action
  -> 处理完成，或者未处理完则唤醒 ksoftirqd
```

和 [[workqueue机制]] 最大的区别是：

| 机制 | 执行上下文 | 能否睡眠 | 常见用途 |
| --- | --- | --- | --- |
| hardirq | 硬中断上下文 | 不能 | 尽快确认硬件事件 |
| softirq | 软中断上下文 | 不能 | 网络收包、定时器、RCU、调度辅助 |
| tasklet | softirq 上下文 | 不能 | 旧式下半部，现代代码里逐渐少用 |
| workqueue | kworker 线程上下文 | 可以 | 可阻塞的异步后台任务 |

一句话：softirq 适合“必须尽快做，但不能放在硬中断里做太久，而且不需要睡眠”的事情。

## 为什么需要 softirq

硬中断处理函数应该非常短。

比如网卡收到包后触发中断。如果在硬中断里直接完成完整协议栈处理，就会产生几个问题：

| 问题 | 后果 |
| --- | --- |
| 硬中断时间太长 | 其他中断响应被拖慢 |
| 当前 CPU 被长时间占住 | 调度延迟和系统抖动增加 |
| 驱动和协议栈逻辑复杂 | hardirq 里更容易写出难维护的同步问题 |

所以 Linux 常用两段式处理：

```text
top half:
  尽快确认硬件事件
  关闭或限流设备中断
  把后续工作标记为 softirq pending
  返回

bottom half:
  在 softirq 中批量处理包、定时器、RCU callback 等
```

softirq 比 workqueue 更轻、更快，因为它不需要把工作交给普通内核线程调度后再执行。代价是它仍然是原子上下文，不能睡眠，也不能做可能长时间阻塞的事情。

## softirq 是固定种类，不是随便创建

workqueue 里可以动态创建很多 `workqueue_struct`。softirq 不一样。

softirq 的类型是内核编译期固定的一组向量。常见枚举大致是：

```c
enum {
        HI_SOFTIRQ = 0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        IRQ_POLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,
        NR_SOFTIRQS
};
```

不同内核版本可能会调整细节，但思想稳定：

| softirq | 大致用途 |
| --- | --- |
| `HI_SOFTIRQ` | 高优先级 tasklet |
| `TIMER_SOFTIRQ` | 普通内核定时器 |
| `NET_TX_SOFTIRQ` | 网络发送完成等处理 |
| `NET_RX_SOFTIRQ` | 网络收包，NAPI poll 的主路径 |
| `BLOCK_SOFTIRQ` | 块层 I/O completion 等下半部处理 |
| `IRQ_POLL_SOFTIRQ` | irq polling 相关处理 |
| `TASKLET_SOFTIRQ` | 普通 tasklet |
| `SCHED_SOFTIRQ` | 调度器辅助，比如负载均衡相关工作 |
| `HRTIMER_SOFTIRQ` | 高精度 timer 的部分处理 |
| `RCU_SOFTIRQ` | RCU callback 处理 |

普通驱动通常不会新增一个 softirq 类型。驱动更常见的是使用已有机制：

| 需求 | 常见选择 |
| --- | --- |
| 网络收包 | NAPI，底层走 `NET_RX_SOFTIRQ` |
| 需要原子上下文下半部 | 旧代码可能用 tasklet，新代码谨慎选择替代机制 |
| 可以睡眠或逻辑较重 | workqueue |
| 需要专属线程和调度策略 | kthread 或 threaded IRQ |

## 核心数据结构

softirq 的核心结构很小。

## `softirq_vec`

内核维护一个全局 softirq action 表：

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS];
```

每个元素对应一种 softirq。

`struct softirq_action` 里最关键的是回调函数：

```c
struct softirq_action {
        void (*action)(struct softirq_action *);
};
```

初始化时，各子系统会注册自己的处理函数：

```c
open_softirq(NET_RX_SOFTIRQ, net_rx_action);
open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
open_softirq(RCU_SOFTIRQ, rcu_core_si);
```

可以这么理解：

```text
softirq_vec[NET_RX_SOFTIRQ].action = net_rx_action
softirq_vec[TIMER_SOFTIRQ].action  = run_timer_softirq
softirq_vec[RCU_SOFTIRQ].action    = rcu_core_si
```

`open_softirq()` 通常只由内核核心子系统在初始化时调用，不是普通模块随手扩展 softirq 类型的接口。

## per-CPU pending 位图

每个 CPU 都有自己的 softirq pending 位图。

当某个 CPU 上 raise 了 `NET_RX_SOFTIRQ`，本质上就是把当前 CPU 的 pending 位图里 `NET_RX_SOFTIRQ` 对应 bit 置上。

可以抽象成：

```text
CPU0 pending: 0000_1000  表示 CPU0 有 NET_RX_SOFTIRQ 待处理
CPU1 pending: 0000_0000  表示 CPU1 暂时没有 pending softirq
```

这个 per-CPU 设计很重要：

| 特点 | 意义 |
| --- | --- |
| 每个 CPU 独立 pending | 减少全局锁竞争 |
| softirq 通常在 raise 它的 CPU 上执行 | 保持缓存局部性 |
| 不同 CPU 可以同时执行同一种 softirq | 并发能力强，但要求处理函数自己设计好同步 |

这也是 softirq 比 tasklet 更“底层”的地方。softirq action 可能在多个 CPU 上并行运行；如果共享全局状态，必须自己加锁或使用 per-CPU 数据结构。

## raise_softirq 做了什么

raise softirq 不是立刻调用处理函数。

它主要做两件事：

```text
raise_softirq(nr)
  -> 关本地中断
  -> __raise_softirq_irqoff(nr)
  -> 打开本地中断
```

`__raise_softirq_irqoff(nr)` 的核心就是：

```text
or_softirq_pending(1 << nr)
```

也就是设置当前 CPU 的 pending bit。

常见 API：

| API | 含义 |
| --- | --- |
| `raise_softirq(nr)` | 置 pending bit，内部会处理本地中断开关 |
| `raise_softirq_irqoff(nr)` | 调用者已经关中断时使用 |
| `__raise_softirq_irqoff(nr)` | 更底层，只置 bit，要求调用者已经满足上下文约束 |

路径大致是：

```text
硬中断 handler
  -> raise_softirq_irqoff(NET_RX_SOFTIRQ)
  -> hardirq return
  -> irq_exit()
  -> invoke_softirq()
  -> __do_softirq()
  -> net_rx_action()
```

注意，raise 只是“登记待处理”。真正什么时候执行，要看当前是否在中断上下文、BH 是否被禁用、是否需要唤醒 `ksoftirqd` 等。

## __do_softirq 主循环

`__do_softirq()` 是 softirq 的核心执行函数。

它做的事情可以概括为：

```text
__do_softirq()
  -> 读取当前 CPU pending 位图
  -> 清空 pending 位图
  -> 对每个置位的 softirq:
       调用 softirq_vec[nr].action()
  -> 如果执行过程中又产生新的 pending:
       在时间和次数允许时继续处理
       否则唤醒 ksoftirqd
```

伪代码大概是：

```c
restart:
        pending = local_softirq_pending();
        set_softirq_pending(0);

        while (pending) {
                nr = next_set_bit(pending);
                h = softirq_vec + nr;
                h->action(h);
                pending &= ~(1 << nr);
        }

        pending = local_softirq_pending();
        if (pending && time_before(jiffies, end) && !need_resched() && --max_restart)
                goto restart;

        if (pending)
                wakeup_softirqd();
```

真实代码比这个复杂，要处理 tracing、lockdep、preempt count、架构栈切换等，但主逻辑就是这个。

## 为什么要限制执行时间

softirq 有一个天然风险：

```text
softirq handler 执行
  -> 又 raise 同类 softirq
  -> __do_softirq() 继续处理
  -> 又产生更多 softirq
  -> ...
```

如果没有限制，CPU 可能长时间困在 softirq 里，普通进程得不到调度。

所以内核会限制一次 `__do_softirq()` 的处理量。主线内核里常见限制包括：

| 限制 | 作用 |
| --- | --- |
| 时间上限 | 避免一次 softirq 执行太久 |
| restart 次数上限 | 即使时间计数异常，也不会无限循环 |
| `need_resched()` 检查 | 如果调度器认为该让出 CPU，就停止继续贪心处理 |

处理不完的 pending softirq 不会丢。内核会唤醒当前 CPU 对应的 `ksoftirqd/N`，让它在线程上下文里继续跑 softirq。

## ksoftirqd

每个 CPU 通常有一个 `ksoftirqd/N` 内核线程。

比如：

```text
ksoftirqd/0
ksoftirqd/1
ksoftirqd/2
...
```

它的作用是兜底处理 softirq：

```text
softirq 太多
  -> hardirq 返回路径不继续跑
  -> wakeup_softirqd()
  -> ksoftirqd/N 被调度
  -> run_ksoftirqd()
  -> __do_softirq()
```

这并不表示 softirq 变成了普通 workqueue。

即使由 `ksoftirqd` 执行，softirq action 仍然按 softirq 规则运行：

| 点 | 说明 |
| --- | --- |
| 仍然不能睡眠 | softirq handler 仍然是 bottom-half 语义 |
| 仍然要短小 | `ksoftirqd` 只是避免硬中断返回路径被拖死 |
| 会受调度器影响 | 既然是线程，就可能被更高优先级任务影响 |

一个常见现象是：系统网络流量很大时，`top` 里能看到 `ksoftirqd/N` CPU 占用升高。这通常说明 softirq 处理压力已经大到不能完全在中断返回路径里消化。

## local_bh_disable 和 local_bh_enable

BH 是 bottom half 的缩写。softirq、tasklet 这类机制都属于 bottom half 语境。

`local_bh_disable()` 的意思不是关闭硬件中断，而是在当前 CPU 上暂时禁止 softirq 执行。

常见用法：

```c
local_bh_disable();
/* 访问会被 softirq 同时访问的数据 */
local_bh_enable();
```

它常用于保护进程上下文和 softirq 上下文共享的数据。

比如某个数据结构既可能在普通系统调用路径里访问，也可能在 `NET_RX_SOFTIRQ` 里访问。普通 spinlock 只能防并发访问，但如果当前进程拿着锁时被本 CPU softirq 打断，而 softirq 也想拿同一把锁，就可能死锁。

这时常见模式是：

```c
spin_lock_bh(&lock);
/* 临界区 */
spin_unlock_bh(&lock);
```

它相当于：

```text
local_bh_disable()
spin_lock()
...
spin_unlock()
local_bh_enable()
```

几个相关 API：

| API | 含义 |
| --- | --- |
| `local_bh_disable()` | 当前 CPU 禁止 softirq 执行 |
| `local_bh_enable()` | 重新允许 softirq；如果有 pending，可能顺便执行 |
| `spin_lock_bh()` | 关 BH 后拿 spinlock |
| `spin_unlock_bh()` | 放锁后开 BH |
| `in_softirq()` | 判断是否处在 softirq 相关上下文 |
| `in_serving_softirq()` | 判断当前是否正在执行 softirq handler |

`local_bh_disable()` 是 per-CPU 的。它不会阻止其他 CPU 上的 softirq 运行。

## softirq 的执行时机

softirq pending 后，可能在几个地方被执行。

## 硬中断返回路径

最典型的是 hardirq 退出时。

```text
硬中断进入
  -> handler raise softirq
  -> 硬中断退出
  -> 如果当前 CPU 有 pending softirq
  -> invoke_softirq()
  -> __do_softirq()
```

这也是 softirq 低延迟的来源。它不一定要等一个线程被调度才开始。

## local_bh_enable

如果进程上下文里之前 `local_bh_disable()`，期间有人 raise 了 softirq，那么 `local_bh_enable()` 重新打开 BH 时也可能触发处理。

```text
local_bh_disable()
  -> 临界区期间 softirq pending
local_bh_enable()
  -> 发现 pending
  -> do_softirq()
```

## ksoftirqd

如果当前不适合直接跑，或者跑了一段后仍有 pending，就唤醒 `ksoftirqd`。

```text
pending softirq
  -> wakeup_softirqd()
  -> 调度到 ksoftirqd/N
  -> __do_softirq()
```

实际路径会受到内核配置影响。比如 PREEMPT_RT 内核会把更多中断和 softirq 处理线程化，以降低实时延迟。

## softirq 和网络收包

网络收包是理解 softirq 最好的例子。

现代网卡驱动通常使用 NAPI。简化路径是：

```text
网卡收到包
  -> 触发硬中断
  -> 驱动 IRQ handler:
       关闭或抑制网卡 RX 中断
       napi_schedule()
  -> ____napi_schedule()
       把 napi_struct 挂到当前 CPU 的 poll_list
       __raise_softirq_irqoff(NET_RX_SOFTIRQ)
  -> hardirq 返回
  -> __do_softirq()
  -> net_rx_action()
       遍历 poll_list
       调用驱动 poll 函数
       从 RX ring 收包
       送入协议栈
       如果预算用完，留待下次继续
       如果处理完，napi_complete_done() 并重新开中断
```

这里有几个关键点：

| 点 | 说明 |
| --- | --- |
| 硬中断只做调度 | 真正批量收包在 `NET_RX_SOFTIRQ` |
| NAPI 有 budget | 防止一次 softirq 无限处理网络包 |
| 高流量下可能进入 polling | 减少频繁硬中断 |
| 处理不完会继续 pending | 后续由 softirq 或 `ksoftirqd` 继续 |

可以把 NAPI 看成网络子系统在 `NET_RX_SOFTIRQ` 上建立的一套批处理框架。

## softirq 和 timer

普通内核 timer 到期后，也会通过 softirq 处理。

简化路径：

```text
硬件时钟中断
  -> 更新 jiffies / tick
  -> 发现 timer 到期
  -> raise TIMER_SOFTIRQ
  -> run_timer_softirq()
  -> 执行到期 timer callback
```

timer callback 运行在 softirq 上下文，所以也不能睡眠。

这点和 `delayed_work` 很容易混：

| 机制 | 到期后在哪里执行回调 |
| --- | --- |
| kernel timer | timer softirq 上下文，不能睡眠 |
| delayed work | timer 到期后 queue work，最终在 kworker 线程上下文执行，可以睡眠 |

所以如果回调里需要睡眠，不要直接用普通 timer callback 做重活，可以让 timer 只负责 queue work。

## softirq 和 tasklet

tasklet 是建立在 softirq 上的一层旧式下半部接口。

普通 tasklet 底层使用 `TASKLET_SOFTIRQ`，高优先级 tasklet 使用 `HI_SOFTIRQ`。

它大致提供了这样的语义：

| 语义 | 说明 |
| --- | --- |
| 同一个 tasklet 不会在多个 CPU 上同时运行 | 比裸 softirq action 更容易用 |
| 不同 tasklet 可以并行 | 仍然要考虑共享数据同步 |
| 不能睡眠 | 因为底层还是 softirq |

老驱动里经常看到 tasklet。新代码里 tasklet 已经不算推荐方向，很多场景更倾向于 threaded IRQ、workqueue、NAPI 或其他更明确的机制。

## softirq 和 threaded IRQ

threaded IRQ 是另一种把中断处理线程化的方式。

典型接口是：

```c
request_threaded_irq(irq, hardirq_fn, thread_fn, flags, name, dev);
```

大致语义：

```text
hardirq_fn:
  快速确认中断
  返回 IRQ_WAKE_THREAD

thread_fn:
  在 irq thread 里执行较重处理
```

和 softirq 对比：

| 机制 | 特点 |
| --- | --- |
| softirq | 延迟低、并发强、不能睡眠、类型固定 |
| threaded IRQ | 每个 IRQ 可有线程处理函数，更像线程上下文，适合驱动里较复杂的中断处理 |
| workqueue | 通用异步任务，不一定和某个 IRQ 强绑定 |

如果驱动的下半部需要睡眠，softirq/tasklet 通常不是好选择。

## 并发语义

softirq 的并发性比很多人直觉里更强。

重要规则：

1. 同一种 softirq 可以在不同 CPU 上同时执行。
2. 同一个 CPU 上 softirq 处理通常不会重入同一个 softirq handler。
3. softirq handler 运行时本地 BH 处于 disabled 状态。
4. `local_bh_disable()` 只影响当前 CPU，不影响其他 CPU。
5. softirq handler 不能睡眠，不能持有可能睡眠的锁。

所以 softirq handler 常见设计是：

| 设计 | 原因 |
| --- | --- |
| 使用 per-CPU 队列 | 减少跨 CPU 锁竞争 |
| 每次处理有 budget | 限制延迟，避免饿死普通任务 |
| 共享数据用 spinlock | softirq 上下文不能睡眠 |
| 进程上下文访问同一数据时用 `spin_lock_bh()` | 防止本 CPU softirq 中断后抢同一把锁 |

不要把 softirq 当成“自动串行执行的后台任务”。它更像一个 per-CPU、高优先级、原子上下文的事件执行层。

## 能不能睡眠

不能。

softirq handler 里不能调用可能睡眠的接口，比如：

| 不适合在 softirq 中做 | 原因 |
| --- | --- |
| `mutex_lock()` | mutex 可能睡眠 |
| `kmalloc(..., GFP_KERNEL)` | `GFP_KERNEL` 可能进入回收并睡眠 |
| 同步文件 I/O | 可能阻塞 |
| 等待 completion / waitqueue | 可能睡眠 |
| 长时间循环处理大量对象 | 会拉高调度延迟 |

可以使用的通常是原子上下文安全的接口，比如：

| 可以考虑 | 注意 |
| --- | --- |
| spinlock | 持锁时间要短 |
| per-CPU 数据 | 注意跨 CPU 汇总 |
| `GFP_ATOMIC` 分配 | 只能应急，失败概率更高 |
| queue work | 把可睡眠部分交给 workqueue |

常见模式是：

```text
softirq:
  快速搬运状态
  做必须立即做的轻量处理
  如果需要阻塞或复杂逻辑，queue_work()
```

## 和 workqueue 怎么选

可以按这个表先判断：

| 问题 | 倾向选择 |
| --- | --- |
| 必须非常快地响应，且不能睡眠 | softirq / NAPI / timer |
| 回调里需要睡眠、拿 mutex、做同步 I/O | workqueue |
| 是网络收包路径 | NAPI，不要自己发明下半部 |
| 是设备 IRQ 的较复杂处理 | threaded IRQ 或 workqueue |
| 是老代码里的轻量下半部 | 可能看到 tasklet，但新代码谨慎使用 |
| 需要长期循环、可控生命周期 | kthread |

一句话：

```text
softirq 解决低延迟、高频、原子上下文的批处理问题；
workqueue 解决可睡眠、可阻塞、普通线程上下文的异步执行问题。
```

## 调试和观察

可以从 `/proc/softirqs` 看各 CPU 的 softirq 计数：

```bash
cat /proc/softirqs
```

输出大概像：

```text
                    CPU0       CPU1
          HI:          0          0
       TIMER:     123456     120001
      NET_TX:       1000        900
      NET_RX:     999999     800000
       BLOCK:      12345      10000
    IRQ_POLL:          0          0
     TASKLET:        100        200
       SCHED:      55555      50000
     HRTIMER:        123        110
         RCU:      77777      76000
```

看这个文件时要注意：

| 现象 | 可能含义 |
| --- | --- |
| `NET_RX` 快速增长 | 网络收包压力大 |
| 某个 CPU 的 `NET_RX` 明显偏高 | RSS、RPS、IRQ affinity 或流量分布可能不均 |
| `TIMER` 很高 | timer/tick 活跃，不一定是问题 |
| `RCU` 很高 | RCU callback 较多，需结合上下文判断 |
| `ksoftirqd/N` CPU 高 | softirq 压力大，硬中断返回路径处理不完 |

常用排查命令：

```bash
cat /proc/softirqs
top -H
ps -eLo pid,tid,comm,psr,pcpu | grep ksoftirqd
cat /proc/interrupts
```

如果开启 ftrace，可以看相关事件。不同内核版本事件名会有差异，常见方向包括：

```text
irq:softirq_raise
irq:softirq_entry
irq:softirq_exit
napi:napi_poll
```

网络问题还经常结合这些信息看：

| 信息 | 用途 |
| --- | --- |
| `/proc/interrupts` | 网卡 IRQ 分布在哪些 CPU |
| `/proc/softirqs` | `NET_RX` 是否集中在少数 CPU |
| `ethtool -S <dev>` | 驱动和网卡队列统计 |
| `ethtool -l <dev>` / `-L` | 查看和设置队列数量 |
| RPS / XPS / IRQ affinity | 调整收发包 CPU 分布 |

## 常见误区

## 误区一：softirq 是软件模拟的中断，所以可以像普通函数一样用

softirq 确实是软件触发的内核执行机制，但它不是普通函数调用。

它有严格上下文限制：不能睡眠、执行时机受中断返回和 BH 状态影响、并发发生在 per-CPU 层面。

## 误区二：raise_softirq 会立刻执行

raise softirq 只是设置 pending bit。

真正执行可能发生在硬中断返回、`local_bh_enable()`、`ksoftirqd` 等路径。写代码时不能假设 `raise_softirq()` 返回前处理已经完成。

## 误区三：有 ksoftirqd，所以 softirq 可以睡眠

不可以。

`ksoftirqd` 是线程，但它执行的是 softirq action。softirq action 仍然要遵守 softirq 上下文规则。需要睡眠就把那部分工作交给 workqueue 或其他线程上下文。

## 误区四：local_bh_disable 会禁止所有 CPU 的 softirq

不会。

它只禁止当前 CPU 的 bottom half 执行。其他 CPU 仍然可以运行 softirq。如果数据会被其他 CPU softirq 访问，还需要正常的锁或 per-CPU 设计。

## 误区五：softirq 比 workqueue 高级，所以优先用 softirq

softirq 不是更通用的 workqueue。

它更底层、更受限，适合少数高频核心路径。普通驱动和模块里，如果没有非常明确的低延迟原子上下文需求，workqueue、threaded IRQ 或现有子系统框架通常更合适。

## 小结

Linux softirq 是硬中断下半部机制。硬中断 handler 只做最小处理，然后通过 `raise_softirq()` 设置当前 CPU 的 pending 位。内核在硬中断返回、`local_bh_enable()` 或 `ksoftirqd` 路径中调用 `__do_softirq()`，扫描 pending 位图并执行 `softirq_vec[nr].action()`。

它的优势是低延迟、开销小、per-CPU 并发能力强；代价是运行在原子上下文，不能睡眠，也不能无限处理。内核用时间、restart 次数和 `need_resched()` 等机制限制一次 softirq 执行量，处理不完时唤醒 `ksoftirqd/N` 继续兜底。

理解 softirq 时最重要的是三件事：第一，raise 只是置 pending，不是同步执行；第二，softirq action 可能在多个 CPU 上并行运行，必须自己处理同步；第三，只要需要睡眠或复杂阻塞逻辑，就应该转给 workqueue、threaded IRQ 或专门内核线程。
