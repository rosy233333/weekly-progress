# ArceOS任务调度调研

时间：2024/3/10

## 概述

ArceOS的任务调度代码位于`modules/axtask`。

与任务调度相关的`cargo feature`：

- `multitask`：启用任务调度。
- `irq`：启用中断。使任务可以使用`sleep`等计时的API
- `preempt`：支持抢占调度
- `sched_fifo`、`sched_rr`、`sched_cfs`：使用不同的调度器进行调度。默认使用FIFO。RR和CFS会同时启用抢占调度。

## Redis的使用情况

使用了`irq`和`multitask`。

## 详细信息

ArceOS的任务用`Task`表示。在`axtask`模块中定义了`TaskInner`，再在不同的`Scheduler`中进行一层包装形成`Task`。

```Rust
pub struct TaskInner {
    id: TaskId,
    name: String,
    is_idle: bool,
    is_init: bool,

    entry: Option<*mut dyn FnOnce()>,
    state: AtomicU8,

    in_wait_queue: AtomicBool,
    #[cfg(feature = "irq")]
    in_timer_list: AtomicBool,

    #[cfg(feature = "preempt")]
    need_resched: AtomicBool,
    #[cfg(feature = "preempt")]
    preempt_disable_count: AtomicUsize,

    exit_code: AtomicI32,
    wait_for_exit: WaitQueue,

    kstack: Option<TaskStack>,
    ctx: UnsafeCell<TaskContext>,

    #[cfg(feature = "tls")]
    tls: TlsArea,
}
```

任务的运行环境为`AxRunQueue`。其是对`Scheduler`的包装，提供了向内部调度器加入任务的接口。同时，增加了运行任务和对当前任务进行操作的功能。

```Rust
impl AxRunQueue {
    pub fn new() -> SpinNoIrq<Self> { ... }

    pub fn add_task(&mut self, task: AxTaskRef) { ... }

    #[cfg(feature = "irq")]
    pub fn scheduler_timer_tick(&mut self) { ... }

    pub fn yield_current(&mut self) { ... }

    pub fn set_current_priority(&mut self, prio: isize) -> bool { ... }

    #[cfg(feature = "preempt")]
    pub fn preempt_resched(&mut self) { ... }

    pub fn exit_current(&mut self, exit_code: i32) -> ! { ... }

    pub fn block_current<F>(&mut self, wait_queue_push: F)
    where
        F: FnOnce(AxTaskRef),
    { ... }

    pub fn unblock_task(&mut self, task: AxTaskRef, resched: bool) { ... }

    #[cfg(feature = "irq")]
    pub fn sleep_until(&mut self, deadline: axhal::time::TimeValue) { ... }

    /// Common reschedule subroutine. If `preempt`, keep current task's time
    /// slice, otherwise reset it.
    fn resched(&mut self, preempt: bool) { ... }

    fn switch_to(&mut self, prev_task: CurrentTask, next_task: AxTaskRef) { ... }
}
```

任务被阻塞时，加入`WaitQueue`中。其在收到其它任务的通知时（或者设定的计时器达到时间时），重新加入`RunQueue`中，实现唤醒。

按照我的理解，`WaitQueue`可以用于互斥有限资源的情况。任务无法获得资源，因此加入`WaitQueue`阻塞。当其它任务释放资源时，它会通知对应的`WaitQueue`，使其唤醒其中的一个任务。

## 修改方案

保留原有的`TaskInner`描述同步任务，新增一种`AsyncTaskInner`作为对`Future`的包装，描述异步任务。建立一个`trait`统一这两种任务的表示。

新增一种`Scheduler`，调用硬件调度器进行任务调度。在该计划中，任务调度器可以调度同步任务（线程）和异步任务（协程）。

    传统的运行时在用户态实现协程，因此协程包含于操作系统线程中。而这里实现的是操作系统级的协程调度与运行，因此将协程与线程并列。

视情况修改`AxRunQueue`（例如，需要增加运行异步任务的代码），尽量少修改。（可能需要把`AxRunQueue`的一些功能放到`Task trait`里，例如任务运行等。因为同步任务和异步任务的运行需要不同的代码）

之后可能还要修改中断相关的内容，从而使用`ats-intc`的中断机制。这方面我还没仔细看。

## 同步任务（线程）和异步任务（协程）的比较

|操作|同步任务的行为/实现原理|异步任务的行为/实现原理|
|-|-|-|
|添加任务|将任务加入调度器|将任务加入调度器|
|时间片耗尽后切换任务|（在非抢占的FIFO调度器中）无|无|
|交出控制权（yield）|运行过程中调用yield函数，最终调用了`RunQueue`的重调度方法|返回`Pending`，交还控制权给Executor后，由Executor进行重调度|
|退出任务|通知所有等待该任务结束的任务；将该任务加入退出队列（为了让垃圾回收任务回收）；通知一个`WAIT_FOR_EXIT`中的的任务（启动垃圾回收任务）；重调度|通过return来正常退出；如果对`Future`的包装里没有需要手动释放的资源的话，直接`drop`就行；重调度|
|阻塞任务|将任务加入`WaitQueue`；重调度|在Reactor处注册Waker（Reactor可能采用类似`WaitQueue`的机制）；返回`Pending`；重调度|
|唤醒任务|将任务加入`RunQueue`|调用Waker，实质也是将任务加入`RunQueue`|
|重调度|将当前任务重新加入`RunQueue`；取出下一运行的任务；切换上下文|当前任务返回；取出下一运行的任务；调用该任务的`poll`（如果运行时本身有一个函数调用栈的话，那连调用栈也不用切换，直接用这个调用栈就行，因为协程退出的时候一定会保持调用栈的干净）

## 任务切换的深入研究

原有的线程切换，其核心函数的功能为：保存当前任务的寄存器上下文，恢复将要运行的任务的寄存器上下文。其中：

- 运行栈的保存和恢复：每个线程的运行栈存储在`TaskInner`中，不会被线程切换影响；线程切换时，只需保存和恢复栈指针寄存器`sp`即可。
- 执行流的保存和恢复：线程切换时，会保存和恢复包含`ret`跳转地址的寄存器`ra`。保存上下文时，`ra`指向该线程的执行流中，调用上下文切换函数的下一条语句。上下文恢复后，在上下文切换函数的结尾执行`ret`指令，就会回到切换后的线程中，执行完上下文切换函数后的下一条指令。

为了使用相同的接口统一线程切换、协程切换、线程-协程间的切换，计划是**将任务切换的核心代码改为连续调用旧任务的`上下文保存`和新任务的`上下文恢复`两个函数**。两个函数的大致功能：

- 对于线程，两个函数分别对应着寄存器保存和恢复的两个过程。
- 对于协程，`上下文保存`函数不需做任何事，因为当协程被切换时，它已经返回了`Pending`或`Ready`，也已经更新了自身`Future`的状态。`上下文恢复`函数则需要运行这个协程。先调用其内部`Future`的`poll`方法，再调用调度器的任务切换方法。这样的设计，能够发挥协程切换低开销的特点。

设计中的问题：

对于线程切换，**执行流的恢复会有问题**。因为保存时，`ra`中保存的`ret`地址是调用`上下文保存`函数后一条指令的地址，也就是调用`上下文恢复`函数的指令。所以，恢复上下文并`ret`后，执行流又会跳到调用`上下文恢复`函数的指令，造成死循环。这个问题比较好解决，**在`上下文保存`函数中，为`ra`寄存器手动加上一个偏移量，以跳过`上下文恢复`函数即可**。（这里有问题，因为可能存在多级调用，不是一进入`上下文保存`函数就开始保存寄存器……试试看能不能把`上下文保存`内部的所有函数调用全给`inline`）

对于协程切换，在这一设计中，**协程的运行实际上是“借用”了上一个线程的运行栈**。理论上并没有问题，因为协程的运行，实际上是调用一个`poll`方法，该方法返回后，再进行任务切换，所以运行栈是平衡的。但因为是在`上下文恢复`函数内运行协程，又在函数内把任务切走，因此这个函数不会返回，而且切换任务的函数也不会返回（因为下一个协程是在函数内部的`上下文恢复`函数中运行的）。从而造成运行栈不平衡。**但这样的不平衡不会影响到使用这个运行栈的线程**，因为线程的`sp`寄存器的值是由保存恢复得来的，不会被协程影响。协程在栈上写的内容会被当作栈顶之外的无意义数据。**但在一种情况下会产生问题：当调度对象只有协程没有线程时**。这种情况下，运行栈不平衡会逐渐累积，造成内存泄漏甚至栈溢出。还有一个问题是，当线程被切走时已经占用了较多运行栈，那协程可用的运行栈就会减少。

所以对协程切换的部分，需要修改方案，**建立一个`static`的运行栈作为协程专用**（且所有协程共用）（对于多核情况，则是每个CPU核心一个协程专用栈）。**在协程的`上下文恢复`函数的开始，将`sp`寄存器指向协程专用栈的栈底**。这样，两个不返回的函数也不会造成运行栈不平衡，解决了上一个方案中的不足，且只引入了一个`static`栈的空间开销和每次协程切换多一条指令的时间开销。