# zfl学长项目阅读笔记

时间：2024.11.25

本次阅读的内容：了解async-os内核中涉及任务管理的各个模块，从而设计共享调度器的移植方案（即：使用相同的任务数据结构与任务队列管理用户态任务和内核态任务，仅有任务切换机制在用户态和内核态不同）

## modules/trampoline

内核态任务的任务运行/切换机制和中断处理机制。

实现`TaskApi`里对任务的各种操作

## modules/task_api

定义对任务的各种操作的接口

## modules/executor

定义了进程和executor，既作为协程/线程的就绪队列、执行器，也作为进程。

但目前，进程中创建的线程/协程放入内核调度器，而非线程自己的调度器。

## modules/taskctx

定义任务TCB、上下文、栈和Waker（使用了percpu）

## 模块依赖关系

```Mermaid
flowchart TD
trampoline --> task_api
trampoline --> taskctx
trampoline --> executor
task_api --> taskctx
executor --> task_api
```

## 大致的移植思路

将taskctx移植到用户态，保持用户态任务和内核态任务的数据结构一致

将trampoline移植到用户态，提供任务切换和运行功能；其中，任务的就绪队列原本有executor提供，但在用户态直接放在trampoline模块中就可以，因为executor的其它功能都与进程有关。

这样的用户态任务调度应该也能符合task_api中定义的接口。不过，要了解一下用户态程序和系统内核是否是分开编译的，为task_api提供两个实现会不会冲突？