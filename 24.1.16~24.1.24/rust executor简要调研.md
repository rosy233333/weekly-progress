# rust executor 简要调研

时间：2024/1/20

Rust 有如下4中常见的executor ：

- [`futures`](https://github.com/rust-lang/futures-rs)：这个库自带了很简单的executor，支持`no-std`
- [`tokio`](https://github.com/tokio-rs/tokio)：提供executor，当使用`#[tokio::main]`时，就隐含引入了`tokio`的executor
- [`async-std`](https://github.com/async-rs/async-std)：提供executor，和 tokio 类似
- [`smol`](https://github.com/smol-rs/smol)：提供async-executor，主要提供了`block_on`

参考资料：

[Rust学习笔记-异步编程(async/await/Future)](https://zhuanlan.zhihu.com/p/611587154)，重点是里面的`executor 调度器`章节

[Rust：构建你自己的block_on()](https://zhuanlan.zhihu.com/p/148064818)

[Rust：构建你自己的executor（执行器）](https://zhuanlan.zhihu.com/p/149740327)


