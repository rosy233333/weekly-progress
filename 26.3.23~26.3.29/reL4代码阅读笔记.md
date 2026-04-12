# reL4代码阅读笔记

## 代码结构

按照[`rel4_kernel/README.md`](https://github.com/rel4team/rel4_kernel/blob/smp_dev/README.md)的描述，使用如下命令下载完整代码：

```bash
mkdir rel4test && cd rel4test
repo init -u https://github.com/rel4team/sel4test-manifest.git
repo sync
```

下载下来的代码分为`kernel`、`projects`、`rel4_kernel`、`tools`四个子模块。`kernel`是seL4的内核源码，`rel4_kernel`是reL4的内核源码。这两个子模块有相互的依赖，构建过程同时需要这两个子模块。

2026.3.28日：下载的`rel4_kernel`的commit id为d9284e15，我在`smp_dev`（仓库主分支）和`fpga_test`（ldh毕设仓库指向的分支）中都没找到这个commit。

## 出入内核的过程（riscv）

进入内核：

`trap_entry`汇编函数在`kernel/src/arch/riscv/traps.S`中声明，其中在保存上下文后分情况调用`c_handle_syscall`、`c_handle_exception`、`c_handle_interrupt`函数。

`rel4_kernel/src/arch/riscv/platform.rs`的`init_cpu`函数将`trap_entry`写入了`stvec`寄存器。

`c_handle_*`系列函数原本在`kernel`中定义，但这些定义被注释掉了。改为使用`rel4_kernel/src/arch/riscv/c_traps.rs`（在`fpga_test`分支中，是`rel4_kernel/src/kernel/c_traps.rs`）中的定义。

因此，进入内核时，先进入seL4定义的trap向量，保存上下文后跳转到reL4定义的处理函数。

进入内核后的上下文保存：在单核情况下，将t0与sscratch交换；在多核情况下，将sp与sscratch交换（sscratch保存内核栈），再将t0存储于(-2\*REGBYTES)(sp)，加载(-1\*REGBYTES)(sp)到t0。之后将上下文保存于t0指向的地址，将sscratch恢复为sp（内核栈指针）。对比返回用户态的代码可知，单核情况下，用户任务的上下文指针存于sscratch中；多核情况下，用户任务的上下文指针存于(-1\*REGBYTES)(sp)处。该方式成立的条件为所有任务切换都需要进内核，因此进内核时的当前任务就是上一次出内核时的当前任务，因此任务的上下文指针有效。

离开内核：在`c_handle_*`函数中调用`rel4_kernel/src/arch/riscv/c_traps.rs`的`restore_user_context`函数，获取当前线程上下文并返回。

多核情况下有一个内核锁，除了处理`INTERRUPT_IPI_0`（`irq_remote_call_ipi`）中断（这是什么？为什么要例外？）以外都需要申请内核锁（`clh_lock_acquire`），使得同一时间只有一个核心能进入内核。

## 任务调度

在[进程调度调研](../26.3.2~26.3.8/进程调度调研.md)中有部分说明。

### 进程和线程调度

在`rel4_kernel`仓库中，在`rel4_kernel/src/syscall/mod.rs:handle_yield`函数中定义了`yield`系统调用。其引用了[`sel4_task`](https://github.com/rel4team/sel4_task/tree/main)仓库（`d9284e15`版本的内核）或`rel4_kernel/src/task_manager`（`fpga_test`版本的内核）中的函数。许多与任务调度相关的函数都放在这个仓库/模块中。

调度器有一个`scheduler_action`状态，暂时不清楚其作用。

用户态任务的切换就是陷入内核后修改“当前任务”变量，因此在返回用户态后会返回修改后的当前任务的上下文。

内核态没有任务上下文之说。内核态执行流不会发生任务切换，也不会发生中断。可以认为是一个内核任务服务所有的用户态进程/线程。

### 协程调度

内核态通过`executor`调度和切换协程。在发生时钟中断和核间中断时，内核调用`coroutine_run_until_blocked`进入`executor`运行协程（[https://github.com/rel4team/rel4_kernel/blob/smp_dev/src/interrupt/handler.rs#L135](https://github.com/rel4team/rel4_kernel/blob/smp_dev/src/interrupt/handler.rs#L135)）。因此，在内核执行的代码分为协程外和协程内两部分。

每个线程对应一个内核`async_syscall_handler`协程。

## 如何适配我的任务调度模块

### 任务模型

reL4当前的任务模型为：用户态有进程/线程和协程，进程和线程平级，协程在线程内运行；内核执行流不属于任何任务，但可能在内核执行流上再运行内核协程。

其中，因为线程切换需要陷入内核，线程的上下文全部以trap上下文的形式保存。

为了修改为任务调度模块的“地址空间-特权级-执行流”任务模型（在reL4的语境中，“地址空间”包括Capability空间），需要修改以下几项：

- 协程与线程平级
- 将协程外的内核执行流改造为trap处理任务
- 相同地址空间的用户态线程切换需要在用户态进行

### 用户态线程上下文

用户态线程从“全部在内核态切换”改为“一部分在用户态切换（主动让出、系统调用），一部分在内核态切换（中断、异常）”，因此保存的上下文也会出现线程上下文和trap上下文两种。需要分情况讨论进行保存和恢复。

### 内核锁、内核任务与经过内核的切换

内核锁对内核任务、经过内核的切换产生的影响：

reL4的当前设计：由于内核锁的存在，再加上内核执行流不会发生中断和线程切换的特性，可以认为是由同一个内核线程服务所有的用户进程/线程。在内核引入协程后，则内核可能发生协程切换。

在我们的实现中，如果将trap处理任务也视为协程，则内核态只有协程没有线程，可以一定程度上简化任务模型。并且，由于内核锁的存在，trap处理任务只需要一个即可。

（不确定）任务调度模块中管理的消息的同步都由任务调度模块负责，因此不需要额外的内核锁保护。因此，可以使经过内核态的任务切换（例如，用户态进程切换）不需要获取内核锁，只有在执行内核协程前需要获取内核锁？
