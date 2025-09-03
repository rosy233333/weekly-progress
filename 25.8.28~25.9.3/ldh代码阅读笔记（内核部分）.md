# ldh代码阅读笔记

仓库链接：[https://github.com/rel4team/rel4_kernel/tree/fpga_test](https://github.com/rel4team/rel4_kernel/tree/fpga_test)

## 问题与回答

### 有哪些改进空间，是否适合作为基础让我继续工作？

#### 队列结构变化

从SPSC改为MPSC，不再是每一对通信双方维护一组发送和接收队列，而是每一个进程（内核也算一个进程）维护自己的接收队列。

需要进行测试，以得知该变化能否带来性能增长。

#### io_uring机制与协程机制更深的整合

在发送异步IPC/系统调用的过程中，任务的状态变化与IPC信息的传递过程有相似之处：

```
任务放入阻塞队列 --------------------> 任务放回就绪队列
请求放入发送队列 --> 接收方处理信息 --> 响应放回接收队列
```

因此，考虑将任务的状态与IPC信息的状态放在一起维护。

（粗浅的想法）例如，在MPSC队列结构的基础上，将每个进程的接收队列直接用作任务就绪队列。而该队列中的元素包含了TCB和IPC信息两部分。executor在循环过程中，取出队列元素，如果是任务则直接执行；如果是IPC则唤醒相应任务执行，且在执行完成后将阻塞在该IPC上的任务（该任务也存储在队列元素中）放回对方的就绪队列。

通过该方式，不再需要独立的dispatcher协程进行任务分发，而是直接将executor用作dispatcher。

该修改可以提升轮询模式下的性能，因为不再需要运行dispatcher协程。但中断模式下，若整个进程都已离线，则仍需要通过中断来唤醒。或许可以将该机制扩展到进程调度，从而实现不需要中断的进程唤醒？

如果按照如上所示的方向做下去，那么ldh的工作就是一个很好的基础。

## 内核态

### async_runtime模块

实现了协程的调度与管理、共享内存中的通信队列、异步系统调用的处理流程。

协程调度和管理：略

通信队列：此处定义了队列及其元素。使用时，由用户申请队列的内核对象，内核通过用户传入的句柄，从对应的队列中接收信息。

```Rust
#[repr(align(4096))]
pub struct NewBuffer {
    pub recv_req_status: AtomicBool,
    pub recv_reply_status: AtomicBool,
    pub req_items: ItemsQueue,
    pub res_items: ItemsQueue,
}

pub struct ItemsQueue {
    buffer: SafeRingBuffer<IPCItem, MAX_ITEM_NUM>,
    // lock: Mutex<()>,
}

#[repr(align(8))]
#[derive(Clone, Copy, Debug)]
pub struct IPCItem {
    pub cid: CoroutineId,
    pub msg_info: u32,
    pub extend_msg: [u16; MAX_IPC_MSG_LEN],
}
```

异步系统调用流程：（据文档所言）每个用户线程对应一个内核协程。（因此，也对应一个`NewBuffer`。）协程中运行`async_syscall_handler`函数，在循环中从队列中取出`IPCItem`，解析其系统调用号和参数，并调用对应的处理函数。在处理完成后，若用户态发送方已不在线，则调用`send_async_syscall_uintr`函数，发送用户态中断通知发送方。

我理解的异步系统调用的作用：使系统调用的发起方和接收方可以位于不同的核心，例如可以抢占低优先级的核心执行系统调用，而原核心执行优先级更高的任务。

## 用户态

### async_runtime模块

也实现了协程的调度与管理、定义了共享内存中的通信队列。但与内核不同，用于处理用户态IPC的任务（dispatcher协程）不在此模块中

### src/async_lib.rs

实现了用户态程序发起系统调用和发起IPC的方式，以及处理用户态IPC的任务（dispatcher协程）。

发起异步系统调用：填写好`IPCItem`后，调用`seL4_Call_with_item`，该函数（在系统调用情景下）会调用`wake_syscall_handler`进入内核唤醒系统调用处理协程。最后，系统调用的处理结果会通过`yield_now().await`的返回值获取？（`wake_syscall_handler`是否不会返回？因为其返回后上层最终会返回`Err(())`。）

此外，有一个与`seL4_Call_with_item`类似的`seL4_Send_with_item`，但似乎未被使用。

`recv_reply_coroutine`和`recv_reply_coroutine_async_syscall`分别用于接收IPC和系统调用的结果。它们首先通过不同方式获取`NewBuffer`，从中取出结果后，通过`recv_reply_coroutine`通过`item.cid`找到对应的协程并唤醒。如果为系统调用，则还需要在唤醒时向被唤醒协程传入`item`。
