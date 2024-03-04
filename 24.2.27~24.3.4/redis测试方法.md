# Redis测试方法

时间：2024/3/4

## Redis介绍

Redis是一个数据存储工具，分为客户端和服务端。

服务端用C语言编写，已经移植到ArceOS上。（而且我已经在riscv版本的ArceOS上成功跑起来了）

客户端可以使用Rust语言的客户端库[redis-rs](https://github.com/redis-rs/redis-rs)，支持异步，但需要`tokio`或`async-std`作为运行时。其还未移植到ArceOS上。

## 使用Redis测试的系统结构

在调度器上运行的任务应该是Redis的客户端还是服务端？我认为是客户端。因为服务端用C语言编写，很难适应Rust的Future机制。而客户端使用`redis-rs`库，则需修改代码使其支持ats-intc协程运行时，与前者相比较为简单，但也有一定难度。

在此前提下，服务端应该运行在ArceOS虚拟机上还是Linux主机上？我认为两者都可以。其一，我们引入Redis进行实验只是为了让每个Future需要一定的处理时间才能返回，这点与Redis服务端运行在哪个设备上关系不大。其二，ArceOS为unikernel系统，一个系统实例只能运行一个软件。因此无法将Redis客户端和服务端运行在同一个ArceOS实例中，只能创建两台虚拟机分别运行。让Redis服务端运行在ArceOS中并不能避免网络传输数据的过程。

## 对`redis-rs`的研究

`redis-rs`中与异步I/O相关的代码位于`redis` crate的`aio`模块中。

`mod.rs`主要定义了两个trait：`RedisRuntime`和`ConnectionLike`。

`RedisRuntime`用于建立TCP连接，而且还要实现`AsyncRead`和`AsyncWrite`两个trait。（这两个trait好像是tokio里的，不知道是否会对移植造成麻烦）

`aio`模块里，对`RedisRuntime`分别有`tokio`和`async-std`版本的实现。

`ConnectionLike`用于向TCP Socket里发送命令并得到结果。它的实现似乎与使用`tokio`还是`async-std`无关。

从功能上推测，`RedisRuntime`在底层，`ConnectionLike`在上层。但它们在代码中的联系还未仔细研究。

