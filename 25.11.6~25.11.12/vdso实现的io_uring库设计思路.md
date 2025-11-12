# vdso实现的io_uring库设计思路

## 设计目的

使用io_uring机制实现用户态和内核态共享的异步IPC和系统调用机制。

需要兼容[设计思路](https://github.com/oscourse-tsinghua/openingreport-lxi2025/blob/main/%E6%80%9D%E8%B7%AF%E8%AE%BE%E8%AE%A1%E6%95%B4%E7%90%86.md)中的多种通信模式（每一对通信双方使用一对队列；每一个通信方使用一条队列；仅服务方使用队列；不使用队列），从而比较它们之间的性能。

## 工作内容

1. 在vDSO调度器中实现Waker机制（目前该机制还缺失）
2. 选择合适的环形无锁队列和队列元素数据结构（可以参考[本文的思路](../25.7.10~25.7.16/vdso_iouring工作计划.md)以及[tokio团队的io_uring库](https://github.com/tokio-rs/io-uring/tree/master)）
3. 实现io_uring库并与vDSO调度器整合

（不知是否需要进行跨特权级/地址空间的实现和测试，若需要则还需进行如下工作：）

1. 在vDSO调度器中实现特权级切换和进程切换（目前缺失）
2. 在vDSO调度器中实现跨地址空间唤醒的Waker机制
3. 将vDSO调度器移植到一个系统中（例如rel4），接管用户态与内核态的调度

## 结构设计

为了兼容多种通信模式，该io_uring机制分为两个crate实现：

1. 队列管理模块：通信者注册/取消注册接收队列；通过id找到对应的队列；队列的push和pop。该部分模块可以放入vDSO。
2. 通信模块：通过队列管理模块申请相应的队列，并建立如上所述的多种通信模式。将队列操作封装为发送与接收消息，并提供接收消息的dispatcher协程等机制。这部分模块本可以在用户和内核间共享，但其接口可能涉及协程调度，因此无法使用vDSO（？），实现为普通模块吧。

## 详细设计

### 队列管理模块

应该可以沿用[本文](../25.7.10~25.7.16/vdso_iouring工作计划.md)的“vdso共享库实现的io_uring”章节。

### 通信模块

使用`IPCEntity`类型代表一个参加IPC的实体。在“每一对通信双方使用一对队列”模式中，一个进程若想与多个进程分别通信，则需要多个`IPCEntity`。在其它模式下，一个`IPCEntity`与一个进程对应。

`IPCEntity`内部保存了一个队列id，其指向队列管理模块中一个已分配给它的队列（以及为该队列创建的`dispatcher`协程）。`IPCEntity`可以与usize类型相互转化，以便于在进程间传输。

若需要与其它进程通信，则需要将自身的`IPCEntity`传递给其它进程，再让其它进程向该`IPCEntity`发消息。

未来，还需设计无需队列，基于协程调度器的`IPCEntity`。它与基于队列的`IPCEntity`使用同样的接口，可相互兼容。

基于协程调度器的`IPCEntity`可以访问当前进程的协程调度器、可以阻塞和创建协程。

#### 1. 通信的建立

- `register_entity() -> Result<IPCEntity>`：为自身注册`IPCEntity`。用于每个`IPCEntity`与一个进程对应的情景。
- `register_entity_pair() -> Result<(IPCEntity, IPCEntity)>`：注册一对`IPCEntity`。用于每一对通信双方使用一对队列的情景。之后，本进程需要将返回的第二个`IPCEntity`传递给对方进程。

#### 2. 发送消息

- `async IPCEntity::send<T>(&self, dst: &IPCEntity, type: usize, data: T)`：发送消息，不等待返回值。`self`为自身的`IPCEntity`，`dst`为对方的`IPCEntity`。
  
如果`dst`为基于队列的`entry`，则在`dst`的队列中构建`SQE`并提交。

如果`dst`为基于协程调度器的`entry`，则根据`type`创建一个相应的处理协程，并传入`data`作为参数。

在“服务端向客户端返回响应”的场景中，如果客户端基于队列，则使用`send`函数发送响应；如果客户端基于协程，则不使用`send`函数，而是直接从`data`中获取客户端正在阻塞的协程，传入结果后唤醒。

#### 3. 接收消息

- `async IPCEntity::recv<T>(&self, type: usize) -> IPCItem<T>`：等待自身的`IPCEntity`接收到类型为`type`的消息。

如果`self`为基于队列的`entry`，则由dispatcher协程接收IPC后，如果接收到了对应`type`的IPC，则唤醒该协程。

如果`self`为基于协程调度器的`entry`，则协程的唤醒由发送方（只能是发送响应的服务端）负责。此处仅为阻塞该协程。

- `async IPCEntity::send_and_recv<T>(&self, dst: &IPCEntity, send_type: usize, data: T, recv_type: usize)`：发送消息并等待响应。
