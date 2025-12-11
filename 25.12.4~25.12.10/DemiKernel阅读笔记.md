# 《The Demikernel Datapath OS Architecture for Microsecond-scale Datacenter Systems》阅读笔记

实现了“Datapath OS”。

- 异构IO库支持
  - 为不同类型的kernel bypass io库提供了统一的高层API
  - 用软件补足了这些库缺失的层级，保证高层API可用
  - 虽然一般情况下，对每个IO库提供独立的Demikernel
  - 但也支持用一个Demikernel管理多种特定的IO库，例如RDMA和SPDK，使它们协同工作，用于需要调用多种IO设备的应用（例如Redis）
- 并发性：支持微秒级调度。
  - 底层采用Rust协程实现。
  - API通过queue_token和wait方法，支持等待某个/某些/所有IO事件，比epoll/select更灵活。（思路与协程类似，更细粒度的阻塞）
  - 复用用户程序的内核线程作为协程的执行环境。用户程序调用wait时，阻塞并重调度到轮询硬件的协程（负责从硬件接收事件）。接收到对应事件后，唤醒并重调度到用户程序所在协程。
  - 协程状态管理：通过一个位确定协程目前阻塞还是就绪。Waker也通过更改该位实现。Executor通过Lemire的算法高效检索就绪的协程。
- 内存管理：支持零拷贝IO，支持UAF（Use after free）保护。
  - 通过更换用户程序的堆分配器，使内存在初始化时即满足io库的不同要求。
  - Rust所有权机制帮助解决了UAF问题。
  - 不足：需要替换用户程序的堆分配器；未提供并行写保护。
- 性能测试
  - 由于Demikernel不是实现新的IO库，而是在已有IO库上加兼容层
  - 因此测试结果中，Demikernel的性能略低于直接使用IO库的性能，不过差距不大
  - 证明兼容层引入的开销可以接受
