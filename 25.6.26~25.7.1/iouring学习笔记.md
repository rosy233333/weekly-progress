# io_uring学习笔记

主要参考文章[https://arthurchiao.art/blog/intro-to-io-uring-zh/](https://arthurchiao.art/blog/intro-to-io-uring-zh/)。对于一些没弄懂的地方，在下面补充说明。

## 三种工作模式

- 中断驱动模式：进程将IO请求提交给块设备后会进入睡眠（D）状态，块设备在处理完IO请求后会触发硬中断，硬中断中会唤醒进程并通知其IO的完成。
- 轮询模式：指**内核对硬件设备** 的轮询。io_uring会额外启动一个内核进程来循环检查IO的完成。
  
    无论是中断模式还是轮询模式，应用更新SQ后均需调用 `io_uring_enter()` 提交IO请求，从而进入内核处理该请求。其中中断和轮询模式的区别体现在**内核与硬件设备** 的交互方式。

- SQ轮询模式：指**内核对SQ（提交队列）** 的轮询。应用更新SQ后，内核轮询线程自动完成，用户不需调用 `io_uring_enter()` 提交IO请求。

## 各个API在实际使用过程中的作用

见[https://www.cnblogs.com/crazymakercircle/p/17149644.html#autoid-h2-10-0-0](https://www.cnblogs.com/crazymakercircle/p/17149644.html#autoid-h2-10-0-0)的`梳理一下io_uring的核心流程`章节

`io_uring_enter`的功能：[https://www.cnblogs.com/crazymakercircle/p/17149644.html#autoid-h2-10-0-0](https://www.cnblogs.com/crazymakercircle/p/17149644.html#autoid-h2-10-0-0)的`IO 提交`章节

提交请求：

![](./65823d17297843f183b38214e651c601.png)

处理结果：

![](./c5d9962a2e244b949d2c1395cc79e597.png)

如果采用非SQ-poll模式，则提交请求时需要系统调用（`io_uring_enter`），处理结果时不需要。如果采用SQ-poll模式，则两个过程都不需要系统调用。
